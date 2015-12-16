# STS System Call Reference

NOTE: As of this writing, STS remains incomplete.  However, sufficient amounts of code exists to give you a crude flavor of what STS will be like in the future.  This chapter discusses STS V1.5 as of the time this chapter was published to LeanPub.

## System Call Compatibility Matrix

The following list of STS system calls covers everything from version 1.0 through to 1.5.  It's primarily of interest to people who studied earlier STS implementations and wish to compare Kestrel-2 versus Kestrel-3 runtime environments.

|System Call|1.0|1.1|1.2|1.5|
|:---------:|:-:|:-:|:-:|:-:|
|getver   |No|No|No|Yes|
|emit     |No|No|No|Yes (2)|
|type     |No|No|No|Yes (2)|
|polkey   |No|No|No|Yes (2)|
|getkey   |No|No|No|Yes (2)|
|fmtmem   |Yes|Yes|Yes|Yes|
|fremem   |Yes|Yes|Yes|Yes|
|getmem   |Yes|Yes|Yes|Yes|
|movmem   |No|No|Yes (1)|Yes|
|setmem   |No|No|No|Yes|
|zermem   |No|No|No|Yes|
|strDup   |No|No|No|Yes|
|strEql   |No|Yes (1)|Yes (1)|Yes|
|open     |Yes|Yes|Yes|Yes|
|close    |Yes|Yes|Yes|Yes|
|read     |Yes|Yes|Yes|Yes|
|write    |No|No|No|Yes (3)|
|filsiz   |No|No|No|Yes|
|seek     |No|No|Yes (1)|Yes|
|unloadseg|Yes|Yes|Yes|Yes|
|loadseg  |Yes|Yes|Yes|Yes|

Notes:

1.  The same capability existed, but went under a different name and/or had a different calling convention.

2.  It remains unclear if these procedures will remain in future STS versions.

3.  Planned for future release.