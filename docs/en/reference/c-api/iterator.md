# Array Iterator API

*New in version 1.6.* 

## Array Iterator

The array iterator encapsulates many of the key features in ufuncs,
allowing user code to support features like output parameters,
preservation of memory layouts, and buffering of data with the wrong
alignment or type, without requiring difficult coding.

This page documents the API for the iterator.
The iterator is named ``NpyIter`` and functions are
named ``NpyIter_*``.

There is an [introductory guide to array iteration](https://numpy.org/devdocs/arrays.nditer.html#arrays-nditer)
which may be of interest for those using this C API. In many instances,
testing out ideas by creating the iterator in Python is a good idea
before writing the C iteration code.

## Simple Iteration Example

The best way to become familiar with the iterator is to look at its
usage within the NumPy codebase itself. For example, here is a slightly
tweaked version of the code for [``PyArray_CountNonzero``](array.html#c.PyArray_CountNonzero), which counts the
number of non-zero elements in an array.

``` c
npy_intp PyArray_CountNonzero(PyArrayObject* self)
{
    /* Nonzero boolean function */
    PyArray_NonzeroFunc* nonzero = PyArray_DESCR(self)->f->nonzero;

    NpyIter* iter;
    NpyIter_IterNextFunc *iternext;
    char** dataptr;
    npy_intp nonzero_count;
    npy_intp* strideptr,* innersizeptr;

    /* Handle zero-sized arrays specially */
    if (PyArray_SIZE(self) == 0) {
        return 0;
    }

    /*
     * Create and use an iterator to count the nonzeros.
     *   flag NPY_ITER_READONLY
     *     - The array is never written to.
     *   flag NPY_ITER_EXTERNAL_LOOP
     *     - Inner loop is done outside the iterator for efficiency.
     *   flag NPY_ITER_NPY_ITER_REFS_OK
     *     - Reference types are acceptable.
     *   order NPY_KEEPORDER
     *     - Visit elements in memory order, regardless of strides.
     *       This is good for performance when the specific order
     *       elements are visited is unimportant.
     *   casting NPY_NO_CASTING
     *     - No casting is required for this operation.
     */
    iter = NpyIter_New(self, NPY_ITER_READONLY|
                             NPY_ITER_EXTERNAL_LOOP|
                             NPY_ITER_REFS_OK,
                        NPY_KEEPORDER, NPY_NO_CASTING,
                        NULL);
    if (iter == NULL) {
        return -1;
    }

    /*
     * The iternext function gets stored in a local variable
     * so it can be called repeatedly in an efficient manner.
     */
    iternext = NpyIter_GetIterNext(iter, NULL);
    if (iternext == NULL) {
        NpyIter_Deallocate(iter);
        return -1;
    }
    /* The location of the data pointer which the iterator may update */
    dataptr = NpyIter_GetDataPtrArray(iter);
    /* The location of the stride which the iterator may update */
    strideptr = NpyIter_GetInnerStrideArray(iter);
    /* The location of the inner loop size which the iterator may update */
    innersizeptr = NpyIter_GetInnerLoopSizePtr(iter);

    nonzero_count = 0;
    do {
        /* Get the inner loop data/stride/count values */
        char* data = *dataptr;
        npy_intp stride = *strideptr;
        npy_intp count = *innersizeptr;

        /* This is a typical inner loop for NPY_ITER_EXTERNAL_LOOP */
        while (count--) {
            if (nonzero(data, self)) {
                ++nonzero_count;
            }
            data += stride;
        }

        /* Increment the iterator to the next inner loop */
    } while(iternext(iter));

    NpyIter_Deallocate(iter);

    return nonzero_count;
}
```

## Simple Multi-Iteration Example

Here is a simple copy function using the iterator.  The ``order`` parameter
is used to control the memory layout of the allocated result, typically
[``NPY_KEEPORDER``](array.html#c.NPY_KEEPORDER) is desired.

``` c
PyObject *CopyArray(PyObject *arr, NPY_ORDER order)
{
    NpyIter *iter;
    NpyIter_IterNextFunc *iternext;
    PyObject *op[2], *ret;
    npy_uint32 flags;
    npy_uint32 op_flags[2];
    npy_intp itemsize, *innersizeptr, innerstride;
    char **dataptrarray;

    /*
     * No inner iteration - inner loop is handled by CopyArray code
     */
    flags = NPY_ITER_EXTERNAL_LOOP;
    /*
     * Tell the constructor to automatically allocate the output.
     * The data type of the output will match that of the input.
     */
    op[0] = arr;
    op[1] = NULL;
    op_flags[0] = NPY_ITER_READONLY;
    op_flags[1] = NPY_ITER_WRITEONLY | NPY_ITER_ALLOCATE;

    /* Construct the iterator */
    iter = NpyIter_MultiNew(2, op, flags, order, NPY_NO_CASTING,
                            op_flags, NULL);
    if (iter == NULL) {
        return NULL;
    }

    /*
     * Make a copy of the iternext function pointer and
     * a few other variables the inner loop needs.
     */
    iternext = NpyIter_GetIterNext(iter, NULL);
    innerstride = NpyIter_GetInnerStrideArray(iter)[0];
    itemsize = NpyIter_GetDescrArray(iter)[0]->elsize;
    /*
     * The inner loop size and data pointers may change during the
     * loop, so just cache the addresses.
     */
    innersizeptr = NpyIter_GetInnerLoopSizePtr(iter);
    dataptrarray = NpyIter_GetDataPtrArray(iter);

    /*
     * Note that because the iterator allocated the output,
     * it matches the iteration order and is packed tightly,
     * so we don't need to check it like the input.
     */
    if (innerstride == itemsize) {
        do {
            memcpy(dataptrarray[1], dataptrarray[0],
                                    itemsize * (*innersizeptr));
        } while (iternext(iter));
    } else {
        /* For efficiency, should specialize this based on item size... */
        npy_intp i;
        do {
            npy_intp size = *innersizeptr;
            char *src = dataptrarray[0], *dst = dataptrarray[1];
            for(i = 0; i < size; i++, src += innerstride, dst += itemsize) {
                memcpy(dst, src, itemsize);
            }
        } while (iternext(iter));
    }

    /* Get the result from the iterator object array */
    ret = NpyIter_GetOperandArray(iter)[1];
    Py_INCREF(ret);

    if (NpyIter_Deallocate(iter) != NPY_SUCCEED) {
        Py_DECREF(ret);
        return NULL;
    }

    return ret;
}
```

## Iterator Data Types

The iterator layout is an internal detail, and user code only sees
an incomplete struct.

- ``NpyIter``

    This is an opaque pointer type for the iterator. Access to its contents
    can only be done through the iterator API.

- ``NpyIter_Type``

    This is the type which exposes the iterator to Python. Currently, no
    API is exposed which provides access to the values of a Python-created
    iterator. If an iterator is created in Python, it must be used in Python
    and vice versa. Such an API will likely be created in a future version.

- ``NpyIter_IterNextFunc``

    This is a function pointer for the iteration loop, returned by
    [``NpyIter_GetIterNext``](#c.NpyIter_GetIterNext).

- ``NpyIter_GetMultiIndexFunc``

    This is a function pointer for getting the current iterator multi-index,
    returned by [``NpyIter_GetGetMultiIndex``](#c.NpyIter_GetGetMultiIndex).

## Construction and Destruction

- [NpyIter](#c.NpyIter)* ``NpyIter_New``([PyArrayObject](types-and-structures.html#c.PyArrayObject)*  *op* , [npy_uint32](dtype.html#c.npy_uint32)  *flags* , [NPY_ORDER](array.html#c.NPY_ORDER)  *order* , [NPY_CASTING](array.html#c.NPY_CASTING)  *casting* , [PyArray_Descr](types-and-structures.html#c.PyArray_Descr)*  *dtype* )

    Creates an iterator for the given numpy array object ``op``.

    Flags that may be passed in ``flags`` are any combination
    of the global and per-operand flags documented in
    [``NpyIter_MultiNew``](#c.NpyIter_MultiNew), except for [``NPY_ITER_ALLOCATE``](#c.NPY_ITER_ALLOCATE).

    Any of the [``NPY_ORDER``](array.html#c.NPY_ORDER) enum values may be passed to ``order``.  For
    efficient iteration, [``NPY_KEEPORDER``](array.html#c.NPY_KEEPORDER) is the best option, and
    the other orders enforce the particular iteration pattern.

    Any of the [``NPY_CASTING``](array.html#c.NPY_CASTING) enum values may be passed to ``casting``.
    The values include [``NPY_NO_CASTING``](array.html#c.NPY_NO_CASTING), [``NPY_EQUIV_CASTING``](array.html#c.NPY_EQUIV_CASTING),
    [``NPY_SAFE_CASTING``](array.html#c.NPY_SAFE_CASTING), [``NPY_SAME_KIND_CASTING``](array.html#c.NPY_SAME_KIND_CASTING), and
    [``NPY_UNSAFE_CASTING``](array.html#c.NPY_UNSAFE_CASTING).  To allow the casts to occur, copying or
    buffering must also be enabled.

    If ``dtype`` isn’t ``NULL``, then it requires that data type.
    If copying is allowed, it will make a temporary copy if the data
    is castable.  If [``NPY_ITER_UPDATEIFCOPY``](#c.NPY_ITER_UPDATEIFCOPY) is enabled, it will
    also copy the data back with another cast upon iterator destruction.

    Returns NULL if there is an error, otherwise returns the allocated
    iterator.

    To make an iterator similar to the old iterator, this should work.

    ``` c
    iter = NpyIter_New(op, NPY_ITER_READWRITE,
                        NPY_CORDER, NPY_NO_CASTING, NULL);
    ```

    If you want to edit an array with aligned ``double`` code,
    but the order doesn’t matter, you would use this.

    ``` c
    dtype = PyArray_DescrFromType(NPY_DOUBLE);
    iter = NpyIter_New(op, NPY_ITER_READWRITE|
                        NPY_ITER_BUFFERED|
                        NPY_ITER_NBO|
                        NPY_ITER_ALIGNED,
                        NPY_KEEPORDER,
                        NPY_SAME_KIND_CASTING,
                        dtype);
    Py_DECREF(dtype);
    ```

- [NpyIter](#c.NpyIter)* ``NpyIter_MultiNew``([npy_intp](dtype.html#c.npy_intp)  *nop* , [PyArrayObject](types-and-structures.html#c.PyArrayObject)**  *op* , [npy_uint32](dtype.html#c.npy_uint32)  *flags* , [NPY_ORDER](array.html#c.NPY_ORDER)  *order* , [NPY_CASTING](array.html#c.NPY_CASTING)  *casting* , [npy_uint32](dtype.html#c.npy_uint32)*  *op_flags* , [PyArray_Descr](types-and-structures.html#c.PyArray_Descr)**  *op_dtypes* )

    Creates an iterator for broadcasting the ``nop`` array objects provided
    in ``op``, using regular NumPy broadcasting rules.

    Any of the [``NPY_ORDER``](array.html#c.NPY_ORDER) enum values may be passed to ``order``.  For
    efficient iteration, [``NPY_KEEPORDER``](array.html#c.NPY_KEEPORDER) is the best option, and the
    other orders enforce the particular iteration pattern.  When using
    [``NPY_KEEPORDER``](array.html#c.NPY_KEEPORDER), if you also want to ensure that the iteration is
    not reversed along an axis, you should pass the flag
    [``NPY_ITER_DONT_NEGATE_STRIDES``](#c.NPY_ITER_DONT_NEGATE_STRIDES).

    Any of the [``NPY_CASTING``](array.html#c.NPY_CASTING) enum values may be passed to ``casting``.
    The values include [``NPY_NO_CASTING``](array.html#c.NPY_NO_CASTING), [``NPY_EQUIV_CASTING``](array.html#c.NPY_EQUIV_CASTING),
    [``NPY_SAFE_CASTING``](array.html#c.NPY_SAFE_CASTING), [``NPY_SAME_KIND_CASTING``](array.html#c.NPY_SAME_KIND_CASTING), and
    [``NPY_UNSAFE_CASTING``](array.html#c.NPY_UNSAFE_CASTING).  To allow the casts to occur, copying or
    buffering must also be enabled.

    If ``op_dtypes`` isn’t ``NULL``, it specifies a data type or ``NULL``
    for each ``op[i]``.

    Returns NULL if there is an error, otherwise returns the allocated
    iterator.

    Flags that may be passed in ``flags``, applying to the whole
    iterator, are:


    - ``NPY_ITER_C_INDEX``

        Causes the iterator to track a raveled flat index matching C
        order. This option cannot be used with [``NPY_ITER_F_INDEX``](#c.NPY_ITER_F_INDEX).

    - ``NPY_ITER_F_INDEX``

        Causes the iterator to track a raveled flat index matching Fortran
        order. This option cannot be used with [``NPY_ITER_C_INDEX``](#c.NPY_ITER_C_INDEX).

    - ``NPY_ITER_MULTI_INDEX``

        Causes the iterator to track a multi-index.
        This prevents the iterator from coalescing axes to
        produce bigger inner loops. If the loop is also not buffered
        and no index is being tracked ( *NpyIter_RemoveAxis*  can be called),
        then the iterator size can be ``-1`` to indicate that the iterator
        is too large. This can happen due to complex broadcasting and
        will result in errors being created when the setting the iterator
        range, removing the multi index, or getting the next function.
        However, it is possible to remove axes again and use the iterator
        normally if the size is small enough after removal.

    - ``NPY_ITER_EXTERNAL_LOOP``

        Causes the iterator to skip iteration of the innermost
        loop, requiring the user of the iterator to handle it.

        This flag is incompatible with [``NPY_ITER_C_INDEX``](#c.NPY_ITER_C_INDEX),
        [``NPY_ITER_F_INDEX``](#c.NPY_ITER_F_INDEX), and [``NPY_ITER_MULTI_INDEX``](#c.NPY_ITER_MULTI_INDEX).

    - ``NPY_ITER_DONT_NEGATE_STRIDES``

        This only affects the iterator when [``NPY_KEEPORDER``](array.html#c.NPY_KEEPORDER) is
        specified for the order parameter.  By default with
        [``NPY_KEEPORDER``](array.html#c.NPY_KEEPORDER), the iterator reverses axes which have
        negative strides, so that memory is traversed in a forward
        direction.  This disables this step.  Use this flag if you
        want to use the underlying memory-ordering of the axes,
        but don’t want an axis reversed. This is the behavior of
        ``numpy.ravel(a, order='K')``, for instance.

    - ``NPY_ITER_COMMON_DTYPE``

        Causes the iterator to convert all the operands to a common
        data type, calculated based on the ufunc type promotion rules.
        Copying or buffering must be enabled.

        If the common data type is known ahead of time, don’t use this
        flag.  Instead, set the requested dtype for all the operands.

    - ``NPY_ITER_REFS_OK``

        Indicates that arrays with reference types (object
        arrays or structured arrays containing an object type)
        may be accepted and used in the iterator.  If this flag
        is enabled, the caller must be sure to check whether
        ``NpyIter_IterationNeedsAPI(iter)`` is true, in which case
        it may not release the GIL during iteration.

    - ``NPY_ITER_ZEROSIZE_OK``

        Indicates that arrays with a size of zero should be permitted.
        Since the typical iteration loop does not naturally work with
        zero-sized arrays, you must check that the IterSize is larger
        than zero before entering the iteration loop.
        Currently only the operands are checked, not a forced shape.

    - ``NPY_ITER_REDUCE_OK``

        Permits writeable operands with a dimension with zero
        stride and size greater than one.  Note that such operands
        must be read/write.

        When buffering is enabled, this also switches to a special
        buffering mode which reduces the loop length as necessary to
        not trample on values being reduced.

        Note that if you want to do a reduction on an automatically
        allocated output, you must use [``NpyIter_GetOperandArray``](#c.NpyIter_GetOperandArray)
        to get its reference, then set every value to the reduction
        unit before doing the iteration loop.  In the case of a
        buffered reduction, this means you must also specify the
        flag [``NPY_ITER_DELAY_BUFALLOC``](#c.NPY_ITER_DELAY_BUFALLOC), then reset the iterator
        after initializing the allocated operand to prepare the
        buffers.

    - ``NPY_ITER_RANGED``

        Enables support for iteration of sub-ranges of the full
        ``iterindex`` range ``[0, NpyIter_IterSize(iter))``.  Use
        the function [``NpyIter_ResetToIterIndexRange``](#c.NpyIter_ResetToIterIndexRange) to specify
        a range for iteration.

        This flag can only be used with [``NPY_ITER_EXTERNAL_LOOP``](#c.NPY_ITER_EXTERNAL_LOOP)
        when [``NPY_ITER_BUFFERED``](#c.NPY_ITER_BUFFERED) is enabled.  This is because
        without buffering, the inner loop is always the size of the
        innermost iteration dimension, and allowing it to get cut up
        would require special handling, effectively making it more
        like the buffered version.

    - ``NPY_ITER_BUFFERED``

        Causes the iterator to store buffering data, and use buffering
        to satisfy data type, alignment, and byte-order requirements.
        To buffer an operand, do not specify the [``NPY_ITER_COPY``](#c.NPY_ITER_COPY)
        or [``NPY_ITER_UPDATEIFCOPY``](#c.NPY_ITER_UPDATEIFCOPY) flags, because they will
        override buffering.  Buffering is especially useful for Python
        code using the iterator, allowing for larger chunks
        of data at once to amortize the Python interpreter overhead.

        If used with [``NPY_ITER_EXTERNAL_LOOP``](#c.NPY_ITER_EXTERNAL_LOOP), the inner loop
        for the caller may get larger chunks than would be possible
        without buffering, because of how the strides are laid out.

        Note that if an operand is given the flag [``NPY_ITER_COPY``](#c.NPY_ITER_COPY)
        or [``NPY_ITER_UPDATEIFCOPY``](#c.NPY_ITER_UPDATEIFCOPY), a copy will be made in preference
        to buffering.  Buffering will still occur when the array was
        broadcast so elements need to be duplicated to get a constant
        stride.

        In normal buffering, the size of each inner loop is equal
        to the buffer size, or possibly larger if
        [``NPY_ITER_GROWINNER``](#c.NPY_ITER_GROWINNER) is specified.  If
        [``NPY_ITER_REDUCE_OK``](#c.NPY_ITER_REDUCE_OK) is enabled and a reduction occurs,
        the inner loops may become smaller depending
        on the structure of the reduction.

    - ``NPY_ITER_GROWINNER``

        When buffering is enabled, this allows the size of the inner
        loop to grow when buffering isn’t necessary.  This option
        is best used if you’re doing a straight pass through all the
        data, rather than anything with small cache-friendly arrays
        of temporary values for each inner loop.

    - ``NPY_ITER_DELAY_BUFALLOC``

        When buffering is enabled, this delays allocation of the
        buffers until [``NpyIter_Reset``](#c.NpyIter_Reset) or another reset function is
        called.  This flag exists to avoid wasteful copying of
        buffer data when making multiple copies of a buffered
        iterator for multi-threaded iteration.

        Another use of this flag is for setting up reduction operations.
        After the iterator is created, and a reduction output
        is allocated automatically by the iterator (be sure to use
        READWRITE access), its value may be initialized to the reduction
        unit.  Use [``NpyIter_GetOperandArray``](#c.NpyIter_GetOperandArray) to get the object.
        Then, call [``NpyIter_Reset``](#c.NpyIter_Reset) to allocate and fill the buffers
        with their initial values.

    - ``NPY_ITER_COPY_IF_OVERLAP``

        If any write operand has overlap with any read operand, eliminate all
        overlap by making temporary copies (enabling UPDATEIFCOPY for write
        operands, if necessary). A pair of operands has overlap if there is
        a memory address that contains data common to both arrays.

        Because exact overlap detection has exponential runtime
        in the number of dimensions, the decision is made based
        on heuristics, which has false positives (needless copies in unusual
        cases) but has no false negatives.

        If any read/write overlap exists, this flag ensures the result of the
        operation is the same as if all operands were copied.
        In cases where copies would need to be made, **the result of the
        computation may be undefined without this flag!**

    Flags that may be passed in ``op_flags[i]``, where ``0 <= i < nop``:


    - ``NPY_ITER_READWRITE``
    - ``NPY_ITER_READONLY``
    - ``NPY_ITER_WRITEONLY``
        Indicate how the user of the iterator will read or write
        to ``op[i]``.  Exactly one of these flags must be specified
        per operand. Using ``NPY_ITER_READWRITE`` or ``NPY_ITER_WRITEONLY``
        for a user-provided operand may trigger  *WRITEBACKIFCOPY`* 
        semantics. The data will be written back to the original array
        when ``NpyIter_Deallocate`` is called.

    - ``NPY_ITER_COPY``
        Allow a copy of ``op[i]`` to be made if it does not
        meet the data type or alignment requirements as specified
        by the constructor flags and parameters.

    - ``NPY_ITER_UPDATEIFCOPY``

        Triggers [``NPY_ITER_COPY``](#c.NPY_ITER_COPY), and when an array operand
        is flagged for writing and is copied, causes the data
        in a copy to be copied back to ``op[i]`` when
        ``NpyIter_Deallocate`` is called.

        If the operand is flagged as write-only and a copy is needed,
        an uninitialized temporary array will be created and then copied
        to back to ``op[i]`` on calling ``NpyIter_Deallocate``, instead of
        doing the unnecessary copy operation.

    - ``NPY_ITER_NBO``
    - ``NPY_ITER_ALIGNED``
    - ``NPY_ITER_CONTIG``

        Causes the iterator to provide data for ``op[i]``
        that is in native byte order, aligned according to
        the dtype requirements, contiguous, or any combination.

        By default, the iterator produces pointers into the
        arrays provided, which may be aligned or unaligned, and
        with any byte order.  If copying or buffering is not
        enabled and the operand data doesn’t satisfy the constraints,
        an error will be raised.

        The contiguous constraint applies only to the inner loop,
        successive inner loops may have arbitrary pointer changes.

        If the requested data type is in non-native byte order,
        the NBO flag overrides it and the requested data type is
        converted to be in native byte order.

    - ``NPY_ITER_ALLOCATE``

        This is for output arrays, and requires that the flag
        [``NPY_ITER_WRITEONLY``](#c.NPY_ITER_WRITEONLY) or [``NPY_ITER_READWRITE``](#c.NPY_ITER_READWRITE)
        be set.  If ``op[i]`` is NULL, creates a new array with
        the final broadcast dimensions, and a layout matching
        the iteration order of the iterator.

        When ``op[i]`` is NULL, the requested data type
        ``op_dtypes[i]`` may be NULL as well, in which case it is
        automatically generated from the dtypes of the arrays which
        are flagged as readable.  The rules for generating the dtype
        are the same is for UFuncs.  Of special note is handling
        of byte order in the selected dtype.  If there is exactly
        one input, the input’s dtype is used as is.  Otherwise,
        if more than one input dtypes are combined together, the
        output will be in native byte order.

        After being allocated with this flag, the caller may retrieve
        the new array by calling [``NpyIter_GetOperandArray``](#c.NpyIter_GetOperandArray) and
        getting the i-th object in the returned C array.  The caller
        must call Py_INCREF on it to claim a reference to the array.

    - ``NPY_ITER_NO_SUBTYPE``

        For use with [``NPY_ITER_ALLOCATE``](#c.NPY_ITER_ALLOCATE), this flag disables
        allocating an array subtype for the output, forcing
        it to be a straight ndarray.

        TODO: Maybe it would be better to introduce a function
        ``NpyIter_GetWrappedOutput`` and remove this flag?

    - ``NPY_ITER_NO_BROADCAST``

        Ensures that the input or output matches the iteration
        dimensions exactly.

    - ``NPY_ITER_ARRAYMASK``

        *New in version 1.7.* 

        Indicates that this operand is the mask to use for
        selecting elements when writing to operands which have
        the [``NPY_ITER_WRITEMASKED``](#c.NPY_ITER_WRITEMASKED) flag applied to them.
        Only one operand may have [``NPY_ITER_ARRAYMASK``](#c.NPY_ITER_ARRAYMASK) flag
        applied to it.

        The data type of an operand with this flag should be either
        [``NPY_BOOL``](dtype.html#c.NPY_BOOL), [``NPY_MASK``](dtype.html#c.NPY_MASK), or a struct dtype
        whose fields are all valid mask dtypes. In the latter case,
        it must match up with a struct operand being WRITEMASKED,
        as it is specifying a mask for each field of that array.

        This flag only affects writing from the buffer back to
        the array. This means that if the operand is also
        [``NPY_ITER_READWRITE``](#c.NPY_ITER_READWRITE) or [``NPY_ITER_WRITEONLY``](#c.NPY_ITER_WRITEONLY),
        code doing iteration can write to this operand to
        control which elements will be untouched and which ones will be
        modified. This is useful when the mask should be a combination
        of input masks.

    - ``NPY_ITER_WRITEMASKED``

        *New in version 1.7.* 

        This array is the mask for all [``writemasked``](https://numpy.org/devdocs/generated/numpy.nditer.html#numpy.nditer)
        operands. Code uses the ``writemasked`` flag which indicates
        that only elements where the chosen ARRAYMASK operand is True
        will be written to. In general, the iterator does not enforce
        this, it is up to the code doing the iteration to follow that
        promise.

        When ``writemasked`` flag is used, and this operand is buffered,
        this changes how data is copied from the buffer into the array.
        A masked copying routine is used, which only copies the
        elements in the buffer for which ``writemasked``
        returns true from the corresponding element in the ARRAYMASK
        operand.

    - ``NPY_ITER_OVERLAP_ASSUME_ELEMENTWISE``

        In memory overlap checks, assume that operands with
        ``NPY_ITER_OVERLAP_ASSUME_ELEMENTWISE`` enabled are accessed only
        in the iterator order.

        This enables the iterator to reason about data dependency,
        possibly avoiding unnecessary copies.

        This flag has effect only if ``NPY_ITER_COPY_IF_OVERLAP`` is enabled
        on the iterator.

- [NpyIter](#c.NpyIter)* ``NpyIter_AdvancedNew``([npy_intp](dtype.html#c.npy_intp)  *nop* , [PyArrayObject](types-and-structures.html#c.PyArrayObject)**  *op* , [npy_uint32](dtype.html#c.npy_uint32)  *flags* , [NPY_ORDER](array.html#c.NPY_ORDER)  *order* , [NPY_CASTING](array.html#c.NPY_CASTING)  *casting* , [npy_uint32](dtype.html#c.npy_uint32)*  *op_flags* , [PyArray_Descr](types-and-structures.html#c.PyArray_Descr)**  *op_dtypes* , int  *oa_ndim* , int**  *op_axes* , [npy_intp](dtype.html#c.npy_intp) const*  *itershape* , [npy_intp](dtype.html#c.npy_intp)  *buffersize* )

    Extends [``NpyIter_MultiNew``](#c.NpyIter_MultiNew) with several advanced options providing
    more control over broadcasting and buffering.

    If -1/NULL values are passed to ``oa_ndim``, ``op_axes``, ``itershape``,
    and ``buffersize``, it is equivalent to [``NpyIter_MultiNew``](#c.NpyIter_MultiNew).

    The parameter ``oa_ndim``, when not zero or -1, specifies the number of
    dimensions that will be iterated with customized broadcasting.
    If it is provided, ``op_axes`` must and ``itershape`` can also be provided.
    The ``op_axes`` parameter let you control in detail how the
    axes of the operand arrays get matched together and iterated.
    In ``op_axes``, you must provide an array of ``nop`` pointers
    to ``oa_ndim``-sized arrays of type ``npy_intp``.  If an entry
    in ``op_axes`` is NULL, normal broadcasting rules will apply.
    In ``op_axes[j][i]`` is stored either a valid axis of ``op[j]``, or
    -1 which means ``newaxis``.  Within each ``op_axes[j]`` array, axes
    may not be repeated.  The following example is how normal broadcasting
    applies to a 3-D array, a 2-D array, a 1-D array and a scalar.

    **Note**: Before NumPy 1.8 ``oa_ndim == 0` was used for signalling that
    that ``op_axes`` and ``itershape`` are unused. This is deprecated and
    should be replaced with -1. Better backward compatibility may be
    achieved by using [``NpyIter_MultiNew``](#c.NpyIter_MultiNew) for this case.

    ``` c
    int oa_ndim = 3;               /* # iteration axes */
    int op0_axes[] = {0, 1, 2};    /* 3-D operand */
    int op1_axes[] = {-1, 0, 1};   /* 2-D operand */
    int op2_axes[] = {-1, -1, 0};  /* 1-D operand */
    int op3_axes[] = {-1, -1, -1}  /* 0-D (scalar) operand */
    int* op_axes[] = {op0_axes, op1_axes, op2_axes, op3_axes};
    ```

    The ``itershape`` parameter allows you to force the iterator
    to have a specific iteration shape. It is an array of length
    ``oa_ndim``. When an entry is negative, its value is determined
    from the operands. This parameter allows automatically allocated
    outputs to get additional dimensions which don’t match up with
    any dimension of an input.

    If ``buffersize`` is zero, a default buffer size is used,
    otherwise it specifies how big of a buffer to use.  Buffers
    which are powers of 2 such as 4096 or 8192 are recommended.

    Returns NULL if there is an error, otherwise returns the allocated
    iterator.

- [NpyIter](#c.NpyIter)* ``NpyIter_Copy``([NpyIter](#c.NpyIter)*  *iter* )

    Makes a copy of the given iterator.  This function is provided
    primarily to enable multi-threaded iteration of the data.

    *TODO* : Move this to a section about multithreaded iteration.

    The recommended approach to multithreaded iteration is to
    first create an iterator with the flags
    [``NPY_ITER_EXTERNAL_LOOP``](#c.NPY_ITER_EXTERNAL_LOOP), [``NPY_ITER_RANGED``](#c.NPY_ITER_RANGED),
    [``NPY_ITER_BUFFERED``](#c.NPY_ITER_BUFFERED), [``NPY_ITER_DELAY_BUFALLOC``](#c.NPY_ITER_DELAY_BUFALLOC), and
    possibly [``NPY_ITER_GROWINNER``](#c.NPY_ITER_GROWINNER).  Create a copy of this iterator
    for each thread (minus one for the first iterator).  Then, take
    the iteration index range ``[0, NpyIter_GetIterSize(iter))`` and
    split it up into tasks, for example using a TBB parallel_for loop.
    When a thread gets a task to execute, it then uses its copy of
    the iterator by calling [``NpyIter_ResetToIterIndexRange``](#c.NpyIter_ResetToIterIndexRange) and
    iterating over the full range.

    When using the iterator in multi-threaded code or in code not
    holding the Python GIL, care must be taken to only call functions
    which are safe in that context.  [``NpyIter_Copy``](#c.NpyIter_Copy) cannot be safely
    called without the Python GIL, because it increments Python
    references.  The ``Reset*`` and some other functions may be safely
    called by passing in the ``errmsg`` parameter as non-NULL, so that
    the functions will pass back errors through it instead of setting
    a Python exception.

    [``NpyIter_Deallocate``](#c.NpyIter_Deallocate) must be called for each copy.

``int NpyIter_RemoveAxis(NpyIter* iter, int axis)``

    Removes an axis from iteration.  This requires that
    [``NPY_ITER_MULTI_INDEX``](#c.NPY_ITER_MULTI_INDEX) was set for iterator creation, and does
    not work if buffering is enabled or an index is being tracked. This
    function also resets the iterator to its initial state.

    This is useful for setting up an accumulation loop, for example.
    The iterator can first be created with all the dimensions, including
    the accumulation axis, so that the output gets created correctly.
    Then, the accumulation axis can be removed, and the calculation
    done in a nested fashion.

    **WARNING**: This function may change the internal memory layout of
    the iterator.  Any cached functions or pointers from the iterator
    must be retrieved again! The iterator range will be reset as well.

    Returns ``NPY_SUCCEED`` or ``NPY_FAIL``.

- int ``NpyIter_RemoveMultiIndex``([NpyIter](#c.NpyIter)*  *iter* )

    If the iterator is tracking a multi-index, this strips support for them,
    and does further iterator optimizations that are possible if multi-indices
    are not needed.  This function also resets the iterator to its initial
    state.

    **WARNING**: This function may change the internal memory layout of
    the iterator.  Any cached functions or pointers from the iterator
    must be retrieved again!

    After calling this function, ``NpyIter_HasMultiIndex(iter)`` will
    return false.

    Returns ``NPY_SUCCEED`` or ``NPY_FAIL``.

- int ``NpyIter_EnableExternalLoop``([NpyIter](#c.NpyIter)*  *iter* )

    If [``NpyIter_RemoveMultiIndex``](#c.NpyIter_RemoveMultiIndex) was called, you may want to enable the
    flag [``NPY_ITER_EXTERNAL_LOOP``](#c.NPY_ITER_EXTERNAL_LOOP).  This flag is not permitted
    together with [``NPY_ITER_MULTI_INDEX``](#c.NPY_ITER_MULTI_INDEX), so this function is provided
    to enable the feature after [``NpyIter_RemoveMultiIndex``](#c.NpyIter_RemoveMultiIndex) is called.
    This function also resets the iterator to its initial state.

    **WARNING**: This function changes the internal logic of the iterator.
    Any cached functions or pointers from the iterator must be retrieved
    again!

    Returns ``NPY_SUCCEED`` or ``NPY_FAIL``.

- int ``NpyIter_Deallocate``([NpyIter](#c.NpyIter)*  *iter* )

    Deallocates the iterator object and resolves any needed writebacks.

    Returns ``NPY_SUCCEED`` or ``NPY_FAIL``.

- int ``NpyIter_Reset``([NpyIter](#c.NpyIter)*  *iter* , char**  *errmsg* )

    Resets the iterator back to its initial state, at the beginning
    of the iteration range.

    Returns ``NPY_SUCCEED`` or ``NPY_FAIL``.  If errmsg is non-NULL,
    no Python exception is set when ``NPY_FAIL`` is returned.
    Instead, *errmsg is set to an error message.  When errmsg is
    non-NULL, the function may be safely called without holding
    the Python GIL.

- int ``NpyIter_ResetToIterIndexRange``([NpyIter](#c.NpyIter)*  *iter* , [npy_intp](dtype.html#c.npy_intp)  *istart* , [npy_intp](dtype.html#c.npy_intp)  *iend* , char**  *errmsg* )

    Resets the iterator and restricts it to the ``iterindex`` range
    ``[istart, iend)``.  See [``NpyIter_Copy``](#c.NpyIter_Copy) for an explanation of
    how to use this for multi-threaded iteration.  This requires that
    the flag [``NPY_ITER_RANGED``](#c.NPY_ITER_RANGED) was passed to the iterator constructor.

    If you want to reset both the ``iterindex`` range and the base
    pointers at the same time, you can do the following to avoid
    extra buffer copying (be sure to add the return code error checks
    when you copy this code).

    ``` c
    /* Set to a trivial empty range */
    NpyIter_ResetToIterIndexRange(iter, 0, 0);
    /* Set the base pointers */
    NpyIter_ResetBasePointers(iter, baseptrs);
    /* Set to the desired range */
    NpyIter_ResetToIterIndexRange(iter, istart, iend);
    ```

    Returns ``NPY_SUCCEED`` or ``NPY_FAIL``.  If errmsg is non-NULL,
    no Python exception is set when ``NPY_FAIL`` is returned.
    Instead, *errmsg is set to an error message.  When errmsg is
    non-NULL, the function may be safely called without holding
    the Python GIL.

- int ``NpyIter_ResetBasePointers``([NpyIter](#c.NpyIter)  *\*iter* , char**  *baseptrs* , char**  *errmsg* )

    Resets the iterator back to its initial state, but using the values
    in ``baseptrs`` for the data instead of the pointers from the arrays
    being iterated.  This functions is intended to be used, together with
    the ``op_axes`` parameter, by nested iteration code with two or more
    iterators.

    Returns ``NPY_SUCCEED`` or ``NPY_FAIL``.  If errmsg is non-NULL,
    no Python exception is set when ``NPY_FAIL`` is returned.
    Instead, *errmsg is set to an error message.  When errmsg is
    non-NULL, the function may be safely called without holding
    the Python GIL.

    *TODO* : Move the following into a special section on nested iterators.

    Creating iterators for nested iteration requires some care.  All
    the iterator operands must match exactly, or the calls to
    [``NpyIter_ResetBasePointers``](#c.NpyIter_ResetBasePointers) will be invalid.  This means that
    automatic copies and output allocation should not be used haphazardly.
    It is possible to still use the automatic data conversion and casting
    features of the iterator by creating one of the iterators with
    all the conversion parameters enabled, then grabbing the allocated
    operands with the [``NpyIter_GetOperandArray``](#c.NpyIter_GetOperandArray) function and passing
    them into the constructors for the rest of the iterators.

    **WARNING**: When creating iterators for nested iteration,
    the code must not use a dimension more than once in the different
    iterators.  If this is done, nested iteration will produce
    out-of-bounds pointers during iteration.

    **WARNING**: When creating iterators for nested iteration, buffering
    can only be applied to the innermost iterator.  If a buffered iterator
    is used as the source for ``baseptrs``, it will point into a small buffer
    instead of the array and the inner iteration will be invalid.

    The pattern for using nested iterators is as follows.

    ``` c
    NpyIter *iter1, *iter1;
    NpyIter_IterNextFunc *iternext1, *iternext2;
    char **dataptrs1;

    /*
    * With the exact same operands, no copies allowed, and
    * no axis in op_axes used both in iter1 and iter2.
    * Buffering may be enabled for iter2, but not for iter1.
    */
    iter1 = ...; iter2 = ...;

    iternext1 = NpyIter_GetIterNext(iter1);
    iternext2 = NpyIter_GetIterNext(iter2);
    dataptrs1 = NpyIter_GetDataPtrArray(iter1);

    do {
        NpyIter_ResetBasePointers(iter2, dataptrs1);
        do {
            /* Use the iter2 values */
        } while (iternext2(iter2));
    } while (iternext1(iter1));
    ```

- int ``NpyIter_GotoMultiIndex``([NpyIter](#c.NpyIter)*  *iter* , [npy_intp](dtype.html#c.npy_intp) const*  *multi_index* )

    Adjusts the iterator to point to the ``ndim`` indices
    pointed to by ``multi_index``.  Returns an error if a multi-index
    is not being tracked, the indices are out of bounds,
    or inner loop iteration is disabled.

    Returns ``NPY_SUCCEED`` or ``NPY_FAIL``.

- int ``NpyIter_GotoIndex``([NpyIter](#c.NpyIter)*  *iter* , [npy_intp](dtype.html#c.npy_intp)  *index* )

    Adjusts the iterator to point to the ``index`` specified.
    If the iterator was constructed with the flag
    [``NPY_ITER_C_INDEX``](#c.NPY_ITER_C_INDEX), ``index`` is the C-order index,
    and if the iterator was constructed with the flag
    [``NPY_ITER_F_INDEX``](#c.NPY_ITER_F_INDEX), ``index`` is the Fortran-order
    index.  Returns an error if there is no index being tracked,
    the index is out of bounds, or inner loop iteration is disabled.

    Returns ``NPY_SUCCEED`` or ``NPY_FAIL``.

- [npy_intp](dtype.html#c.npy_intp) ``NpyIter_GetIterSize``([NpyIter](#c.NpyIter)*  *iter* )

    Returns the number of elements being iterated.  This is the product
    of all the dimensions in the shape.  When a multi index is being tracked
    (and  *NpyIter_RemoveAxis*  may be called) the size may be ``-1`` to
    indicate an iterator is too large.  Such an iterator is invalid, but
    may become valid after  *NpyIter_RemoveAxis*  is called. It is not
    necessary to check for this case.

- [npy_intp](dtype.html#c.npy_intp) ``NpyIter_GetIterIndex``([NpyIter](#c.NpyIter)*  *iter* )

    Gets the ``iterindex`` of the iterator, which is an index matching
    the iteration order of the iterator.

- void ``NpyIter_GetIterIndexRange``([NpyIter](#c.NpyIter)*  *iter* , [npy_intp](dtype.html#c.npy_intp)*  *istart* , [npy_intp](dtype.html#c.npy_intp)*  *iend* )

    Gets the ``iterindex`` sub-range that is being iterated.  If
    [``NPY_ITER_RANGED``](#c.NPY_ITER_RANGED) was not specified, this always returns the
    range ``[0, NpyIter_IterSize(iter))``.

- int ``NpyIter_GotoIterIndex``([NpyIter](#c.NpyIter)*  *iter* , [npy_intp](dtype.html#c.npy_intp)  *iterindex* )

    Adjusts the iterator to point to the ``iterindex`` specified.
    The IterIndex is an index matching the iteration order of the iterator.
    Returns an error if the ``iterindex`` is out of bounds,
    buffering is enabled, or inner loop iteration is disabled.

    Returns ``NPY_SUCCEED`` or ``NPY_FAIL``.

- [npy_bool](dtype.html#c.npy_bool) ``NpyIter_HasDelayedBufAlloc``([NpyIter](#c.NpyIter)*  *iter* )

    Returns 1 if the flag [``NPY_ITER_DELAY_BUFALLOC``](#c.NPY_ITER_DELAY_BUFALLOC) was passed
    to the iterator constructor, and no call to one of the Reset
    functions has been done yet, 0 otherwise.

- [npy_bool](dtype.html#c.npy_bool) ``NpyIter_HasExternalLoop``([NpyIter](#c.NpyIter)*  *iter* )

    Returns 1 if the caller needs to handle the inner-most 1-dimensional
    loop, or 0 if the iterator handles all looping. This is controlled
    by the constructor flag [``NPY_ITER_EXTERNAL_LOOP``](#c.NPY_ITER_EXTERNAL_LOOP) or
    [``NpyIter_EnableExternalLoop``](#c.NpyIter_EnableExternalLoop).

- [npy_bool](dtype.html#c.npy_bool) ``NpyIter_HasMultiIndex``([NpyIter](#c.NpyIter)*  *iter* )

    Returns 1 if the iterator was created with the
    [``NPY_ITER_MULTI_INDEX``](#c.NPY_ITER_MULTI_INDEX) flag, 0 otherwise.

- [npy_bool](dtype.html#c.npy_bool) ``NpyIter_HasIndex``([NpyIter](#c.NpyIter)*  *iter* )

    Returns 1 if the iterator was created with the
    [``NPY_ITER_C_INDEX``](#c.NPY_ITER_C_INDEX) or [``NPY_ITER_F_INDEX``](#c.NPY_ITER_F_INDEX)
    flag, 0 otherwise.

- [npy_bool](dtype.html#c.npy_bool) ``NpyIter_RequiresBuffering``([NpyIter](#c.NpyIter)*  *iter* )

    Returns 1 if the iterator requires buffering, which occurs
    when an operand needs conversion or alignment and so cannot
    be used directly.

- [npy_bool](dtype.html#c.npy_bool) ``NpyIter_IsBuffered``([NpyIter](#c.NpyIter)*  *iter* )

    Returns 1 if the iterator was created with the
    [``NPY_ITER_BUFFERED``](#c.NPY_ITER_BUFFERED) flag, 0 otherwise.

- [npy_bool](dtype.html#c.npy_bool) ``NpyIter_IsGrowInner``([NpyIter](#c.NpyIter)*  *iter* )

    Returns 1 if the iterator was created with the
    [``NPY_ITER_GROWINNER``](#c.NPY_ITER_GROWINNER) flag, 0 otherwise.

- [npy_intp](dtype.html#c.npy_intp) ``NpyIter_GetBufferSize``([NpyIter](#c.NpyIter)*  *iter* )

    If the iterator is buffered, returns the size of the buffer
    being used, otherwise returns 0.

- int ``NpyIter_GetNDim``([NpyIter](#c.NpyIter)*  *iter* )

    Returns the number of dimensions being iterated.  If a multi-index
    was not requested in the iterator constructor, this value
    may be smaller than the number of dimensions in the original
    objects.

- int ``NpyIter_GetNOp``([NpyIter](#c.NpyIter)*  *iter* )

    Returns the number of operands in the iterator.

- [npy_intp](dtype.html#c.npy_intp)* ``NpyIter_GetAxisStrideArray``([NpyIter](#c.NpyIter)*  *iter* , int  *axis* )

    Gets the array of strides for the specified axis. Requires that
    the iterator be tracking a multi-index, and that buffering not
    be enabled.

    This may be used when you want to match up operand axes in
    some fashion, then remove them with ``NpyIter_RemoveAxis`` to
    handle their processing manually.  By calling this function
    before removing the axes, you can get the strides for the
    manual processing.

    Returns ``NULL`` on error.

- int ``NpyIter_GetShape``([NpyIter](#c.NpyIter)*  *iter* , [npy_intp](dtype.html#c.npy_intp)*  *outshape* )

    Returns the broadcast shape of the iterator in ``outshape``.
    This can only be called on an iterator which is tracking a multi-index.

    Returns ``NPY_SUCCEED`` or ``NPY_FAIL``.

- [PyArray_Descr](types-and-structures.html#c.PyArray_Descr)** ``NpyIter_GetDescrArray``([NpyIter](#c.NpyIter)*  *iter* )

    This gives back a pointer to the ``nop`` data type Descrs for
    the objects being iterated.  The result points into ``iter``,
    so the caller does not gain any references to the Descrs.

    This pointer may be cached before the iteration loop, calling
    ``iternext`` will not change it.

- [PyObject](https://docs.python.org/dev/c-api/structures.html#c.PyObject)** ``NpyIter_GetOperandArray``([NpyIter](#c.NpyIter)*  *iter* )

    This gives back a pointer to the ``nop`` operand PyObjects
    that are being iterated.  The result points into ``iter``,
    so the caller does not gain any references to the PyObjects.

- [PyObject](https://docs.python.org/dev/c-api/structures.html#c.PyObject)* ``NpyIter_GetIterView``([NpyIter](#c.NpyIter)*  *iter* , [npy_intp](dtype.html#c.npy_intp)  *i* )

    This gives back a reference to a new ndarray view, which is a view
    into the i-th object in the array [``NpyIter_GetOperandArray``](#c.NpyIter_GetOperandArray),
    whose dimensions and strides match the internal optimized
    iteration pattern.  A C-order iteration of this view is equivalent
    to the iterator’s iteration order.

    For example, if an iterator was created with a single array as its
    input, and it was possible to rearrange all its axes and then
    collapse it into a single strided iteration, this would return
    a view that is a one-dimensional array.

- void ``NpyIter_GetReadFlags``([NpyIter](#c.NpyIter)*  *iter* , char*  *outreadflags* )

    Fills ``nop`` flags. Sets ``outreadflags[i]`` to 1 if
    ``op[i]`` can be read from, and to 0 if not.

- void ``NpyIter_GetWriteFlags``([NpyIter](#c.NpyIter)*  *iter* , char*  *outwriteflags* )

    Fills ``nop`` flags. Sets ``outwriteflags[i]`` to 1 if
    ``op[i]`` can be written to, and to 0 if not.

- int ``NpyIter_CreateCompatibleStrides``([NpyIter](#c.NpyIter)*  *iter* , [npy_intp](dtype.html#c.npy_intp)  *itemsize* , [npy_intp](dtype.html#c.npy_intp)*  *outstrides* )

    Builds a set of strides which are the same as the strides of an
    output array created using the [``NPY_ITER_ALLOCATE``](#c.NPY_ITER_ALLOCATE) flag, where NULL
    was passed for op_axes.  This is for data packed contiguously,
    but not necessarily in C or Fortran order. This should be used
    together with [``NpyIter_GetShape``](#c.NpyIter_GetShape) and [``NpyIter_GetNDim``](#c.NpyIter_GetNDim)
    with the flag [``NPY_ITER_MULTI_INDEX``](#c.NPY_ITER_MULTI_INDEX) passed into the constructor.

    A use case for this function is to match the shape and layout of
    the iterator and tack on one or more dimensions.  For example,
    in order to generate a vector per input value for a numerical gradient,
    you pass in ndim*itemsize for itemsize, then add another dimension to
    the end with size ndim and stride itemsize.  To do the Hessian matrix,
    you do the same thing but add two dimensions, or take advantage of
    the symmetry and pack it into 1 dimension with a particular encoding.

    This function may only be called if the iterator is tracking a multi-index
    and if [``NPY_ITER_DONT_NEGATE_STRIDES``](#c.NPY_ITER_DONT_NEGATE_STRIDES) was used to prevent an axis
    from being iterated in reverse order.

    If an array is created with this method, simply adding ‘itemsize’
    for each iteration will traverse the new array matching the
    iterator.

    Returns ``NPY_SUCCEED`` or ``NPY_FAIL``.

- [npy_bool](dtype.html#c.npy_bool) ``NpyIter_IsFirstVisit``([NpyIter](#c.NpyIter)*  *iter* , int  *iop* )

    *New in version 1.7.* 

    Checks to see whether this is the first time the elements of the
    specified reduction operand which the iterator points at are being
    seen for the first time. The function returns a reasonable answer
    for reduction operands and when buffering is disabled. The answer
    may be incorrect for buffered non-reduction operands.

    This function is intended to be used in EXTERNAL_LOOP mode only,
    and will produce some wrong answers when that mode is not enabled.

    If this function returns true, the caller should also check the inner
    loop stride of the operand, because if that stride is 0, then only
    the first element of the innermost external loop is being visited
    for the first time.

    *WARNING* : For performance reasons, ‘iop’ is not bounds-checked,
    it is not confirmed that ‘iop’ is actually a reduction operand,
    and it is not confirmed that EXTERNAL_LOOP mode is enabled. These
    checks are the responsibility of the caller, and should be done
    outside of any inner loops.

## Functions For Iteration

- [NpyIter_IterNextFunc](#c.NpyIter_IterNextFunc)* ``NpyIter_GetIterNext``([NpyIter](#c.NpyIter)*  *iter* , char**  *errmsg* )

    Returns a function pointer for iteration.  A specialized version
    of the function pointer may be calculated by this function
    instead of being stored in the iterator structure. Thus, to
    get good performance, it is required that the function pointer
    be saved in a variable rather than retrieved for each loop iteration.

    Returns NULL if there is an error.  If errmsg is non-NULL,
    no Python exception is set when ``NPY_FAIL`` is returned.
    Instead, *errmsg is set to an error message.  When errmsg is
    non-NULL, the function may be safely called without holding
    the Python GIL.

    The typical looping construct is as follows.

    ``` c
    NpyIter_IterNextFunc *iternext = NpyIter_GetIterNext(iter, NULL);
    char** dataptr = NpyIter_GetDataPtrArray(iter);

    do {
        /* use the addresses dataptr[0], ... dataptr[nop-1] */
    } while(iternext(iter));
    ```

    When [``NPY_ITER_EXTERNAL_LOOP``](#c.NPY_ITER_EXTERNAL_LOOP) is specified, the typical
    inner loop construct is as follows.

    ``` c
    NpyIter_IterNextFunc *iternext = NpyIter_GetIterNext(iter, NULL);
    char** dataptr = NpyIter_GetDataPtrArray(iter);
    npy_intp* stride = NpyIter_GetInnerStrideArray(iter);
    npy_intp* size_ptr = NpyIter_GetInnerLoopSizePtr(iter), size;
    npy_intp iop, nop = NpyIter_GetNOp(iter);

    do {
        size = *size_ptr;
        while (size--) {
            /* use the addresses dataptr[0], ... dataptr[nop-1] */
            for (iop = 0; iop < nop; ++iop) {
                dataptr[iop] += stride[iop];
            }
        }
    } while (iternext());
    ```

    Observe that we are using the dataptr array inside the iterator, not
    copying the values to a local temporary.  This is possible because
    when ``iternext()`` is called, these pointers will be overwritten
    with fresh values, not incrementally updated.

    If a compile-time fixed buffer is being used (both flags
    [``NPY_ITER_BUFFERED``](#c.NPY_ITER_BUFFERED) and [``NPY_ITER_EXTERNAL_LOOP``](#c.NPY_ITER_EXTERNAL_LOOP)), the
    inner size may be used as a signal as well.  The size is guaranteed
    to become zero when ``iternext()`` returns false, enabling the
    following loop construct.  Note that if you use this construct,
    you should not pass [``NPY_ITER_GROWINNER``](#c.NPY_ITER_GROWINNER) as a flag, because it
    will cause larger sizes under some circumstances.

    ``` c
    /* The constructor should have buffersize passed as this value */
    #define FIXED_BUFFER_SIZE 1024

    NpyIter_IterNextFunc *iternext = NpyIter_GetIterNext(iter, NULL);
    char **dataptr = NpyIter_GetDataPtrArray(iter);
    npy_intp *stride = NpyIter_GetInnerStrideArray(iter);
    npy_intp *size_ptr = NpyIter_GetInnerLoopSizePtr(iter), size;
    npy_intp i, iop, nop = NpyIter_GetNOp(iter);

    /* One loop with a fixed inner size */
    size = *size_ptr;
    while (size == FIXED_BUFFER_SIZE) {
        /*
        * This loop could be manually unrolled by a factor
        * which divides into FIXED_BUFFER_SIZE
        */
        for (i = 0; i < FIXED_BUFFER_SIZE; ++i) {
            /* use the addresses dataptr[0], ... dataptr[nop-1] */
            for (iop = 0; iop < nop; ++iop) {
                dataptr[iop] += stride[iop];
            }
        }
        iternext();
        size = *size_ptr;
    }

    /* Finish-up loop with variable inner size */
    if (size > 0) do {
        size = *size_ptr;
        while (size--) {
            /* use the addresses dataptr[0], ... dataptr[nop-1] */
            for (iop = 0; iop < nop; ++iop) {
                dataptr[iop] += stride[iop];
            }
        }
    } while (iternext());
    ```

- [NpyIter_GetMultiIndexFunc](#c.NpyIter_GetMultiIndexFunc) *``NpyIter_GetGetMultiIndex``([NpyIter](#c.NpyIter)*  *iter* , char**  *errmsg* )

    Returns a function pointer for getting the current multi-index
    of the iterator.  Returns NULL if the iterator is not tracking
    a multi-index.  It is recommended that this function
    pointer be cached in a local variable before the iteration
    loop.

    Returns NULL if there is an error.  If errmsg is non-NULL,
    no Python exception is set when ``NPY_FAIL`` is returned.
    Instead, *errmsg is set to an error message.  When errmsg is
    non-NULL, the function may be safely called without holding
    the Python GIL.

- char** ``NpyIter_GetDataPtrArray``([NpyIter](#c.NpyIter)*  *iter* )

    This gives back a pointer to the ``nop`` data pointers.  If
    [``NPY_ITER_EXTERNAL_LOOP``](#c.NPY_ITER_EXTERNAL_LOOP) was not specified, each data
    pointer points to the current data item of the iterator.  If
    no inner iteration was specified, it points to the first data
    item of the inner loop.

    This pointer may be cached before the iteration loop, calling
    ``iternext`` will not change it.  This function may be safely
    called without holding the Python GIL.

- char** ``NpyIter_GetInitialDataPtrArray``([NpyIter](#c.NpyIter)*  *iter* )

    Gets the array of data pointers directly into the arrays (never
    into the buffers), corresponding to iteration index 0.

    These pointers are different from the pointers accepted by
    ``NpyIter_ResetBasePointers``, because the direction along
    some axes may have been reversed.

    This function may be safely called without holding the Python GIL.

- [npy_intp](dtype.html#c.npy_intp)* ``NpyIter_GetIndexPtr``([NpyIter](#c.NpyIter)*  *iter* )

    This gives back a pointer to the index being tracked, or NULL
    if no index is being tracked.  It is only useable if one of
    the flags [``NPY_ITER_C_INDEX``](#c.NPY_ITER_C_INDEX) or [``NPY_ITER_F_INDEX``](#c.NPY_ITER_F_INDEX)
    were specified during construction.

    When the flag [``NPY_ITER_EXTERNAL_LOOP``](#c.NPY_ITER_EXTERNAL_LOOP) is used, the code
    needs to know the parameters for doing the inner loop.  These
    functions provide that information.

- [npy_intp](dtype.html#c.npy_intp)* ``NpyIter_GetInnerStrideArray``([NpyIter](#c.NpyIter)*  *iter* )

    Returns a pointer to an array of the ``nop`` strides,
    one for each iterated object, to be used by the inner loop.

    This pointer may be cached before the iteration loop, calling
    ``iternext`` will not change it. This function may be safely
    called without holding the Python GIL.

    **WARNING**: While the pointer may be cached, its values may
    change if the iterator is buffered.

- [npy_intp](dtype.html#c.npy_intp)* ``NpyIter_GetInnerLoopSizePtr``([NpyIter](#c.NpyIter)*  *iter* )

    Returns a pointer to the number of iterations the
    inner loop should execute.

    This address may be cached before the iteration loop, calling
    ``iternext`` will not change it.  The value itself may change during
    iteration, in particular if buffering is enabled.  This function
    may be safely called without holding the Python GIL.

- void ``NpyIter_GetInnerFixedStrideArray``([NpyIter](#c.NpyIter)*  *iter* , [npy_intp](dtype.html#c.npy_intp)*  *out_strides* )

    Gets an array of strides which are fixed, or will not change during
    the entire iteration.  For strides that may change, the value
    NPY_MAX_INTP is placed in the stride.

    Once the iterator is prepared for iteration (after a reset if
    ``NPY_DELAY_BUFALLOC`` was used), call this to get the strides
    which may be used to select a fast inner loop function.  For example,
    if the stride is 0, that means the inner loop can always load its
    value into a variable once, then use the variable throughout the loop,
    or if the stride equals the itemsize, a contiguous version for that
    operand may be used.

    This function may be safely called without holding the Python GIL.

## Converting from Previous NumPy Iterators

The old iterator API includes functions like PyArrayIter_Check,
PyArray_Iter* and PyArray_ITER_*.  The multi-iterator array includes
PyArray_MultiIter*, PyArray_Broadcast, and PyArray_RemoveSmallest.  The
new iterator design replaces all of this functionality with a single object
and associated API.  One goal of the new API is that all uses of the
existing iterator should be replaceable with the new iterator without
significant effort. In 1.6, the major exception to this is the neighborhood
iterator, which does not have corresponding features in this iterator.

Here is a conversion table for which functions to use with the new iterator:

Iterator Functions |  Return type
---|---
[PyArray_IterNew](array.html#c.PyArray_IterNew) | [NpyIter_New](#c.NpyIter_New)
[PyArray_IterAllButAxis](array.html#c.PyArray_IterAllButAxis) | [NpyIter_New](#c.NpyIter_New) + ``axes`` parameter or Iterator flag [NPY_ITER_EXTERNAL_LOOP](#c.NPY_ITER_EXTERNAL_LOOP)
[PyArray_BroadcastToShape](array.html#c.PyArray_BroadcastToShape) | NOT SUPPORTED (Use the support for multiple operands instead.)
[PyArrayIter_Check](array.html#c.PyArrayIter_Check) | Will need to add this in Python exposure
[PyArray_ITER_RESET](array.html#c.PyArray_ITER_RESET) | [NpyIter_Reset](#c.NpyIter_Reset)
[PyArray_ITER_NEXT](array.html#c.PyArray_ITER_NEXT) | Function pointer from [NpyIter_GetIterNext](#c.NpyIter_GetIterNext)
[PyArray_ITER_DATA](array.html#c.PyArray_ITER_DATA) | [NpyIter_GetDataPtrArray](#c.NpyIter_GetDataPtrArray)
[PyArray_ITER_GOTO](array.html#c.PyArray_ITER_GOTO) | [NpyIter_GotoMultiIndex](#c.NpyIter_GotoMultiIndex)
[PyArray_ITER_GOTO1D](array.html#c.PyArray_ITER_GOTO1D) | [NpyIter_GotoIndex](#c.NpyIter_GotoIndex) or [NpyIter_GotoIterIndex](#c.NpyIter_GotoIterIndex)
[PyArray_ITER_NOTDONE](array.html#c.PyArray_ITER_NOTDONE) | Return value of ``iternext`` function pointer
*Multi-iterator Functions* |  
[PyArray_MultiIterNew](array.html#c.PyArray_MultiIterNew) | [NpyIter_MultiNew](#c.NpyIter_MultiNew)
[PyArray_MultiIter_RESET](array.html#c.PyArray_MultiIter_RESET) | [NpyIter_Reset](#c.NpyIter_Reset)
[PyArray_MultiIter_NEXT](array.html#c.PyArray_MultiIter_NEXT) | Function pointer from [NpyIter_GetIterNext](#c.NpyIter_GetIterNext)
[PyArray_MultiIter_DATA](array.html#c.PyArray_MultiIter_DATA) | [NpyIter_GetDataPtrArray](#c.NpyIter_GetDataPtrArray)
[PyArray_MultiIter_NEXTi](array.html#c.PyArray_MultiIter_NEXTi) | NOT SUPPORTED (always lock-step iteration)
[PyArray_MultiIter_GOTO](array.html#c.PyArray_MultiIter_GOTO) | [NpyIter_GotoMultiIndex](#c.NpyIter_GotoMultiIndex)
[PyArray_MultiIter_GOTO1D](array.html#c.PyArray_MultiIter_GOTO1D) | [NpyIter_GotoIndex](#c.NpyIter_GotoIndex) or [NpyIter_GotoIterIndex](#c.NpyIter_GotoIterIndex)
[PyArray_MultiIter_NOTDONE](array.html#c.PyArray_MultiIter_NOTDONE) | Return value of ``iternext`` function pointer
[PyArray_Broadcast](array.html#c.PyArray_Broadcast) | Handled by [NpyIter_MultiNew](#c.NpyIter_MultiNew)
[PyArray_RemoveSmallest](array.html#c.PyArray_RemoveSmallest) | Iterator flag [NPY_ITER_EXTERNAL_LOOP](#c.NPY_ITER_EXTERNAL_LOOP)
Other Functions |  
[PyArray_ConvertToCommonType](array.html#c.PyArray_ConvertToCommonType) | Iterator flag [NPY_ITER_COMMON_DTYPE](#c.NPY_ITER_COMMON_DTYPE)