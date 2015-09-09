bui# Summary

* [Buffer](buffer/buffer.md)
	* [重要特性](buffer/important_features.md)
        * [Pooled buffer](buffer/pooled_buffer.md)
        * [Reference Count](buffer/reference_count.md)
            * [接口ReferenceCounted](buffer/interface_ReferenceCounted.md)
            * [类AbstractReferenceCountedByteBuf代码实现](buffer/class_AbstractReferenceCountedByteBuf.md)
            * [buffer的释放](buffer/buffer_release.md)
            * [buffer泄露检测](buffer/lack_detection.md)
            * [类ResourceLeakDetector代码实现](buffer/class_ResourceLeakDetector.md)
        * [Zero Copy](buffer/zero_copy.md)
            * [类SlicedByteBuf代码实现](buffer/class_SlicedByteBuf.md)
            * [方法Unpooled.wrappedBuffer()代码实现](buffer/unpooled_wapped.md)
	* [代码分析](buffer/code_analyze.md)





