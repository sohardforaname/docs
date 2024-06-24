# Postgresql的nbtree索引数据结构

索引结构定义

B-linked tree：
一个N阶平衡树，无论是叶子节点还是内部节点都有链接右兄弟的节点，在节点满的时候会把一部分数据转移到兄弟节点中，仅两个节点都满了才会触发分裂。
其余情况和B-tree相同，即data存key值，为它指向的页的最小值（或者最大值），只有在叶子节点上，才会存储所有的key。

一个B-linked tree的page组成分为：
0号page存储index的元数据。剩余的page存储索引数据。

创建一个空的page之后，最开始操作是insert，从insert开始解读nbtree索引。

入口函数：nbtree.c:btinsert;
总结一下执行流
fn btinsert:
    tuple := index_form_tuple
    key := makekey
    check contains null
    fn _bt_search_insert:
        fetch page
        check if in rightmost leaf page:
            use fast insert
        fn _bt_search:
            page := get root page
            while not leaf:
                page := need move right(page)
                create stack node
                page := child page(according to key to run binsearch)
            page := need move right(page)

    // get a page stack that ok to insert.
    // top is the target page

    if check unique:
        xwait := call fn _bt_check_unique
        if xwait is valid:
            wait other txn to finish insertion and retry
    
    if not check unique exists:
        check serializable conflict
        find inserting location in page and insert

    release page

其中比较重要的是 _bt_check_unique，即唯一性校验
fn _bt_check_unique:
    offset := find page offset by binsearch
    inposting := false
    nbuf := invalid buffer
    loop:
        if offset <= maxoff:
            // fastpath 
            if nbuf is invalid and offset is strict high:
                break
            if inposting or item is alive:
                if item is alive but not inposting:
                    get index tuple or break if pass all equal-tuple
                tuple := get heap tuple from index
                
                if check ok:
                    set found tag
                else:
                    tuple := get tuple from HOT
                    check need wait and return xid

                    if tuple all dead:
                        set index tuple delete tag
            
        tuple := advance to next alive tuple
        if tuple is isvalid:
            page := advance to next page

    if check and not found:
        error: failed to re-find

    return invalid xid

insert完成后需要做的是get/scan
先看btgettuple:
fn btgettuple(scan):
    do:
        if scan not init:
            call _bt_first for the first value
        else:
            if last is killed:
                record killed item
            call _bt_next for the next value
    while keys > 0 and _bt_start_prim_scan // misunderstand currently

fn _bt_first:
    preprocess_keys, set the smallest range of scan

    if parallel_scan:
        // ignored
    else:
        set up scan key
    
    determine scan strategy and start keys by condition '>'
    
    select a scan function for single-index and multi-index.
    
    determine limitation comparation rules.

    call _bt_search to get the page of first tuple

    call _bt_binsrch to get the tuple offset

    read page
    return

fn _bt_next:
    advance tuple according to direction

    if tuple position out of range

    get the next page and unpin current page
    return

清理操作
入口函数nbtree.c:btvacuumscan函数
这个函数执行的是针对这个btree的每一个page执行vacuum操作，除了meta page，具体的函数在btvacuumpage中。
fn btvacuumpage:
    // get buf, as comment saying, we should get page even if it's all-zero page.
    buf := ReadBufferExtended()

    if not all-zero page:
        check page
    
    if cur_block != scan_block:
        // two cases:
        1. the page split after start scan
        2. the index is currupted
    
    check page and if it's leaf:
        upgrade to write lock
            loop all of the index tuple in the page:
                if not posting:
                    callback to vacuum and record to deletable array
                else:
                    res := call btvacuumposting
                    // in btvacuumposting, we also find posting list and
                    // callback to record.


            




            
            



            

        
