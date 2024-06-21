+++
title = 'Buffer pool in a database'
date = 2024-06-21T00:18:33+05:30

+++
In this post we'll try to explore what a Bufferpool is and why its needed in a database.

Any database we use needs to access file on disk either to read or write data. Disk access is slow. So we have to try to minimize this disk access.

*How can we minimize disk access ?* \
We can cache the data in memory. Reads/writes can be done in memory and the in-memory data can be flushed to disk when required.

The Bufferpool is an in-memory cache of buffer-pages to store data read from disk. 

*Before we move ahead:* 
- A file on disk can be imagined to be made of **blocks** and memory can be imagined to be made of **pages**. The block-size and page-size will be equal. 
- A **client** is someone who wants to access a disk-block to either read or write data. 


*When a client wants to access a disk-block the following steps take place*
- The client sends a request to buffer manager. 
- The buffer manager selects a buffer-page from the Bufferpool. 
- The contents of the disk-block is read(if needed) to the selected buffer-page and the page is returned to the client. At this point the buffer-page is said to be **pinned** to the disk-block by the client. 
- The client reads/writes data to the buffer-page(in-memory)
- Once its usage is done, the client requests the buffer manager to **unpin** the buffer-page

The buffer manager does not immediately flush the buffer-page to disk. It holds the data in the buffer-page until it explicitly receives a request to flush the data or till the buffer-page has to be replaced with the contents of another disk-block. 


**When a client requests a buffer manager for accessing a disk-block, there can be \
4 cases**

- *A buffer-page holding the contents of the disk-block is present in Bufferpool and* 

    1. *The buffer-page is pinned* \
    This case is when another client is currently accessing the same buffer-page. \
    A page can be pinned by multiple clients, so the buffer manager simply adds another pin to the page and returns the page to the client
    
    2. *The buffer-page is unpinned* \
    This case is when another client had earlier accessed the buffer-page but its usage was done and it requested the buffer manager to unpin. So the buffer-page is not being accessed by any client currently. \
    Since the contents of the block are still in the buffer page, the buffer manager can reuse the page by simply pinning it and returning it to the client.
    
- *A buffer-page holding the contents of the disk-block is not present in Bufferpool and*

    3. *There is atleast one unpinned buffer-page in the Bufferpool* \
    The buffer manager must first select a buffer-page from the list of unpinned buffers. Second, if the selected buffer-page has been modified, then the buffer manager must flush the page contents to disk.\
    Finally, the requested block can be read into the selected page, and the page can be pinned

    4. *There is no unpinned buffer-page available* \
    The client can either wait till a buffer-page becomes available or request again at a later time.



**Implementation**

- The first 2 cases above can be handled by using a hash map that maps a block to a buffer-page.

If a client requests access to a disk-block, we can check the map and return the page if a corresponding buffer-page is present.\
When a buffer-page is first allocated to a disk-block, we add it to the map. \
When a buffer-page holding contents of a disk-block is made to hold the contents of a new disk-block, we remove the mapping for the old block and add mapping for the new block.

- The third case is where we have to pick a buffer-page from the list of unpinned buffer-pages.
This can be done using LRU, LFU and other strategies.

LRU can be implemented by using a list of unpinned buffers.
When a buffer's pin count becomes 0 (no longer used by any client), add it to the tail-end of the list.
When a buffer needs to be chosen, the Least Recently Used buffer-page is present at the head of the list so remove the buffer-page at the head of the list and use it.



This post is based on the implementation of buffer pool in this project.
https://github.com/naveen246/kite-db