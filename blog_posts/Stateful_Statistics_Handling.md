Clearwater Stateful Statistics Handling
---------------------------------------
We have written before about how Chronos is used in Clearwater to provide [stateful statistics](Stateful_Statistics.md), and the recent enhancements we made to the [Chronos API](Chronos_API_Changes.md) to better handle the tags that drive these statistics. As we noted in these posts, Chronos is able to dynamically create statistics for any new tag it sees, and as such it does not need to maintain a list of expected tags, leaving the statistics free of being tied to any particular version of Clearwater. There were two key requirements we had in mind when designing this feature:

*   There should not be a pre-defined tag list
*   Any node should be able to respond to any valid SNMP request, regardless of whether it has seen that tag before

This post is going to take a closer look at the structures we have designed that enable this functionality and their SNMP interface, as well as the interactions on the Chronos side when timers are sent in.

## The Infinite Table

The core functionality of the dynamic, tag-based statistics creation has been built into a component we have called the ‘Infinite’ table. It has been named this because from the point of view of SNMP it appears to be infinite. i.e. SNMP can query any OID under the OID each table is registered to, and it will receive a value; if a real value is found under the table it is returned, if not a default value of 0 is returned. The code for the `snmp_infinite_base_table`, and for the two subclasses we use described below, is all found in [cpp-common](https://github.com/Metaswitch/cpp-common/tree/master/src).

### GET requests

Each instance of the infinite table is registered to a root OID (e.g. `.1.2.826.0.1.1578918.999`), and the table class passes the OID of any requests under this root OID to internal parsing functions. These functions then attempt to parse the requested OID by assuming requests have the following format: `<root OID>.<Tag Length>.<Tag as . separated ASCII code>.<snmp table column>.<snmp table row>` For example, the request for column 2, row 1 of the table for tag "TAG" would look as follows: `<root OID>.3.84.65.71.2.1` The internal functions can then work along this OID, from left to right, parsing each section as it goes, resulting in a basic logic as follows:

*   Move to the end of the root OID
*   Check the tag length is valid (1 - 16)
*   Move along the next numbers, parsing them off as ASCII values to determine the tag.
*   Check the final two values for the table column and row.
*   Set up a default return value of 0, and then attempt to get a new value from the underlying data structure for the parsed tag.
*   Return the value to net_snmp.

### GETNEXT requests

If the request is an `SNMP GETNEXT`, the table functions will first attempt to find the next valid OID. The key logic of this process is as follows:

*   Check that the tag length has been provided, and is not 0 or greater than the maximum allowed tag length.
*   Check the tag for any characters greater than 'Z'.
*   Check the tag for any characters less than 'A'.
*   Ensure that the number of tag characters matches the expected tag length.
*   Check for provided column and row values, ensure they are valid, and increment accordingly.

This logic is performed within an infinite loop, with the possibility to break out at any of the stages. This enables us to alter a value within the OID, and then return to the beginning of the logic processing until we are satisfied that the processing is complete.

*   Tag length
    *   If the tag length was not provided or is 0, we skip the table and return the OID after incrementing its last digit.
    *   If the tag length was greater than our maximum value, we skip the table and return the OID after incrementing its last digit.
*   Tag characters
    *   If any character is found to be greater than 'Z', we increment the character before it. If the character is the first in the tag, we increment the tag length. At this point we also discard all tag characters after the character just incremented. We then return to the top of the loop.
    *   If any character is found to be less than 'A', we set it, and all characters following it for the given length of the tag, to be 'A'. The column and row values are set to point to the first cell of the table under this new tag, and we break out of the whole loop.
*   Column and Row
    *   If we have not broken out of the loop already, we check the column and row values.
    *   If no value was provided for the column, we assume the first non-index column, and set the two values to point to the first valid cell, before breaking out of the loop.
    *   If the column value is too large, we increment the last character of the tag, and return to the top of the loop.
    *   If no value was provided for the row, we assume the first row, set the correct value, and break out of the loop.
    *   If the row value is too large, we increment the column value, and return to the top of the loop.
    *   If both a column and row value are provided, and they are not too large, we can safely increment the row number and break out of the loop.

Once we have broken out of the loop, we are left with the next valid OID after the OID provided. Through the above logic, the infinite table is able to handle any SNMP query that is passed to it. Even if the tag has never been seen before the table is still able to parse it, check the underlying data structure to see if any values exist for the specified tag, and return 0 or any value found.

### Scalar vs Timer Count Tables

The statistics themselves are stored using two subclasses of the `snmp_infinite_base_table`: the `snmp_infinite_scalar_table` and the `snmp_infinite_timer_count_table`. The key difference between the two subclasses is the structure used beneath to store the data. The scalar table simply stores a single `uint32_t` value for each tag, as it is used to provide the instantaneous value for the stored data. The timer count table instead stores statistics using the `timer_counter` class, which calculates and stores the average, variance, and high and low water marks for the count of each tag for each of the following time periods:

*   Previous five seconds
*   Current five minutes
*   Previous five minutes

Both of the tables simply expose increment and decrement interfaces to Chronos.

## Chronos interactions

As detailed in the [API documentation](https://github.com/Metaswitch/chronos/blob/master/doc/api.md), Chronos timer requests can be sent with a `"statistics"` object in the JSON body, specifying tags to be used in reporting statistics. These tags are parsed from the JSON, and stored as a map of type to count. The tags are then passed through to the statistics tables by the update_statistics function in the timer handler.

### Tags

The tags are stored as part of the timer object so that comparisons can be made on timer updates to see how the statistics have been altered. This also ensures that tag information is maintained as timers replicate across nodes.

### Tables

Currently we maintain two statistical views of the tags passed in as part of timers: an instantaneous count, and a time averaged set of statistics. These statistics are held in the infinite tables discussed further below. One instance of both the `infinite_scalar_table` and the `infinite_timer_count_table` are instantiated at start up. When Chronos receives a timer, we perform logic in the `update_statistics` function to determine if any statistics should be updated.

### Update Statistics

When Chronos receives a timer to add, it first attempts to find any version of the timer within the store. If Chronos decides that it should insert the new timer into the store:

*   If there is not an already existing timer, the timer handler passes the new tags to the `update_statistics` function.
*   If the new timer will overwrite an existing timer, the timer handler passes both the new and the old tags to the `update_statistics` function.
*   If the new timer being added is a tombstone overwriting an existing timer, only the old tags are passed to the `update_statistics` function, as tombstones should not count towards our statistics.

Within `update_statistics` the following logic process is followed to build up a view of what tags are being added, and what tags are being removed:

*   Iterate over the tags associated with the old timer. For each tag:
    *   See if the tag is present in the tags associated with the new timer
    *   If it is not present, add the tag and its count value to the map of tags-to-remove
*   Iterate over the tags associated with the new timer. For each tag:
    *   If the tag is not present in the old tags, add it and its count value to the list of tags-to-add
    *   If the tag is present in the old tags, and the count value is higher in the new tags, add the tag and the difference between the two counts to the list of tags-to-add
    *   If the tag is present in the old tags, and the count value is lower in the new tags, add the tag and the difference between the two counts to the list of tags-to-remove

Within Chronos, each table exposes an increment and a decrement function, which take a tag string, and a value to increment or decrement by. With the two views of tags-to-add and tags-to-remove calculated, we can simply move over each, and increment for the tags-to-add or decrement for the tags-to-remove accordingly. Detail on what statistics are exposed in Project Clearwater, and how to access them, can be found [here,](https://clearwater.readthedocs.io/en/stable/Clearwater_SNMP_Statistics/index.html) with specific information on these stateful statistics [here.](https://clearwater.readthedocs.io/en/stable/Clearwater_Stateful_Statistics/index.html)
