# claujson-document
# first check!
    01. 64bit only
    02. need fast memory allocator like mimalloc, for speed.
    03. C++14~ 
    04. experimental!
    05. Array has std::vector<_Value>
    06. Object has std::vector<Pair<_Value, _Value>> (not a std::map!)
    07. in Object, key can be dupplicated, are not sorted, just in order of input.
    08. scanning - modified? simdjson, parsing - parallel 
    09. it is not read-only! 
    10. _Value <- json_Value, (in destructor, no remove data(Array or Object!), no copy, only move or clone!
    11. Value <- wrap _Value, (in destructor, remove data(Array or Object!)
    12. Array, Object <- in destructor, remove data.
# parallel parse
# parallel write
