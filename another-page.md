---
author:
- |
    root\
    email: [hmsjwzb@yahoo.com](hmsjwzb@yahoo.com)
bibliography:
- 'sample.bib'
title: 'How bio works?'
---

Overview
========

bio is core data structure of generic block layer in Linux. The Linux
kernel serve the bio request to meet the request of user space
application. Let’s take a overlook about bio.\
\
How bio is handled as follows:

1.  Filesystem use submit\_bio to issue a bio request.

2.  submit\_bio will do some basic accounting on bio. It may increase
    the number of sectors it has served.Then, it will handle the bio to\
    generic\_make\_request function.

3.  generic\_make\_request do some basic check on bio and deliver it to\
    blk\_mq\_make\_request.[@It; @maybe; @recursively; @called.; @It; @handles; @the; @current; @request; @and; @previous; @request.]

4.  blk\_mq\_make\_request will do some change on bio. It may split or
    merge bio
    request.[@As; @we; @know; @hardware; @can't; @handle; @all; @unreasonable; @request.]
    Then, it may flush the requests to scheduler’s queue and wake up the
    worker
    thread.[@It; @may; @just; @buffer; @that; @request; @sometime.]

5.  The worker thread invoke blk\_mq\_run\_work\_fn to serve requests.
    It invoke ioscheduler method to get the request the device driver
    will handle next and dispatch it to device driver.

6.  the device driver will handle this request at last.

Core data structure
===================

Let’s look some macros in kernel.

    #define bio_for_each_bvec(bvl, bio, iter)
            __bio_for_each_bvec(bvl, bio, iter, (bio)->bi_iter)

\[lst:bioforeach\] is a loop which traverse every bvec\_iter and
bi\_io\_vec member of bio. Actually bvec\_iter is an iterator. It marks
which part of bio we are processing now.You can refer \[bio\] for
detail.\
In every loop, we construct the bio\_vec data structure. It contains
which page group we are processing and how long we are processing.\

    struct bio_vec {
            struct page     *bv_page;
            unsigned int    bv_len;
            unsigned int    bv_offset;
    };

bio
---

You can find bio definition in

-   It contains the page information about data we are processing.

-   It contains the hardware information we are processed right now.

Core function
=============

generic\_make\_request
----------------------

1.  Get the request\_queue from
    bio.[@request_queue; @is; @core; @data; @structure; @of; @block; @device; @driver.]

2.  Analysis bio request flag and set the appropriate flag. If the
    BIO\_QUEUE\_ENTERED is not marked, the function report error and
    return.

3.  Do some common check on the bio. If the check fails, it returns.

4.  Save the bio\_list\_on\_stack on
    current-&gt;bio\_list.[@bio_list_on_stack[0]; @contains; @bios; @submitted; @by; @current; @make_request_fn; @bio_list_on_stack[1]; @contains; @bio; @submitted; @by; @previous; @make_request_fn]

5.  Create a fresh bio\_list for all subordinate.[@comments]

6.  Invoke blk\_mq\_make\_request handle that bio.

7.  merge queues and restart loop handle the remain bio.

blk\_mq\_make\_request
----------------------

1.  Analysis the flag of the request.(eg. sync request? flush request?)

2.  For non-isa bounce case, just check if the bounce pfn is equal to or
    bigger than the highest pfn in the system.[@comments]

3.  For some hardware limit, we need to split the bio. You can reference
    \[split\]

4.  Try to merge the bio with existence requests on current’s plugged
    list if appropriate.

5.  Merge the request with existence requests in ioscheduler if
    appropriate.

6.  Allocate request for the
    bio.[@The; @allocation; @of; @request; @is; @from; @a; @per-device; @bitmap]

7.  Init the allocated request.

8.  The following code branches according the request type and hardware
    queue numbers.[@We; @mainly; @focus; @on; @one; @queue; @scenario.]

9.  Init request from bio.

10. If the number of requests on current’s plug queue is larger than
    BLK\_MAX\_REQUEST\_COUNT[@16] or the last request size larger than
    BLK\_PLUG\_FLUSH\_SIZE[@128*1024], then we flush the requests on
    current’s plug queue to ioscheduler.

blk\_queue\_split {#split}
-----------------

blk\_queue\_split mainly deal with the work of split bio. The bio is
categorized according to the request type(REQ\_OP\_DISCARD,
REQ\_OP\_SECURE\_ERASE, REQ\_OP\_WRITE\_ZEROS, REQ\_OP\_WRITE\_SAME and
others). The main of our scenario falls into others. It will invoke
blk\_bio\_segment\_split to handle this case. blk\_bio\_segment\_split
use the following loop to handle split.

1.  Check if adding a bio\_vec after bprv with offset would create a gap
    in the SG list. Most drivers don’t care about this, but some
    do.[@comments]

2.  If the sum of current bio\_vec sectors and previous sectors is
    larger than the max\_sectors, then we split the
    bio.[@max_sectors; @is; @determined; @by; @hardware.]

3.  We don’t split the first bio\_vec which is less than the page size.

4.  Multi-page bvec may be too big to hold in one segment, so the
    current bvec has to be splitted as multiple segments.[@comments]

5.  After we split the bio, set the corresponding bi\_seg\_front\_size
    of the new bio.

6.  If we do split, return new bio. Otherwise, return NULL.

After the invoking of blk\_bio\_segment\_split, the blk\_queue\_split
will set the bi\_phys\_segments in split bio and set BIO\_SEG\_VALID and
REQ\_NOMERGE flag for it. Set the corresponding bi\_end\_io method. At
last, it will invoke the generic\_make\_request recursively that queue
the remaining bio in the queue of generic\_make\_request. We will deal
with it next.

bio\_split
----------

The bio\_split will allocate a new bio and split fixed length from the
old bio. The main process is as follow:

1.  Allocate a new bio and copy the original bio information into new
    bio.

2.  Set the length of split bio.

3.  Advance the iterator in old bio.

4.  Set the split bio flag according original bio.

5.  Return the split bio.

blk\_flush\_plug\_list
----------------------

-   blk\_flush\_plug\_list flush the requests in current plug list to
    ioscheduler.

Here is the main process of blk\_mq\_flush\_plug\_list:

1.  Do some basic init.

2.  The while loop take requests from plug list, inset into ioscheduler
    via blk\_mq\_sched\_insert\_requests

3.  Init the hardware context.

4.  invoke the worker thread handle the requests.

blk\_mq\_run\_work\_fn
======================

blk\_mq\_run\_work\_fn is invoked by the worker thread of generic block
layer. It take the blk\_mq\_hw\_ctx from the argument and pass it to
\_\_blk\_mq\_run\_hw\_queue.\
The \_\_blk\_mq\_run\_hw\_queue do some basic checking and invoke
blk\_mq\_sched\_dispatch\_requests to dispatch the requests.\
Here is the main process of blk\_mq\_sched\_dispatch\_requests

1.  Get the data structure we will use.

2.  Grap the previous requests insert into rq\_list

3.  Dispatch the requests.

blk\_mq\_do\_dispatch\_sched
----------------------------

1.  Grap request from ioscheduler.

2.  Insert into rq\_list.

3.  Deliver them to driver via blk\_mq\_dispatch\_rq\_list

bio\_endio
==========

When driver done the work for application, it need to wake up the slept
application.[@Application; @may; @not; @sleep; @on; @some; @circumstance.]
The block device driver will invoke blk\_mq\_complete\_request on the
served request. It will invoke \_\_blk\_mq\_complete\_request
correspondingly. For a single queue device driver, It will call
\_\_blk\_complete\_request for it. For multi-queue,
q-&gt;mq\_ops-&gt;complete will be invoked. The
\_\_blk\_complete\_request will trigger the BLOCK\_SOFTIRQ on
corresponding cpu which the request was served.

[back](./)
