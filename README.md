# claujson-document
# first check!
    1. 64bit only
    2. need fast memory allocator like mimalloc, for speed.
    3. C++14~ 
    4. experimental!
    5. Array has std::vector<_Value>
    6. Object has std::vector<Pair<_Value, _Value>> (not a std::map!)
    7. in Object, key can be dupplicated, are not sorted, just in order of input.
    8. scanning - modified? simdjson, parsing - parallel 
    9. it is not read-only! 
# parallel parse
# parallel write
