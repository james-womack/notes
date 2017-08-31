```flow
st=>start: Start copying PoolA into PoolB (Merge).
obDevViewFromPoolA=>operation: Obtain device pointers for copy.
obHostViewFromPoolA=>operation: Obtain host pointers for copy.
shouldTakeFromHost=>condition: Last level ran on CPU?
copyOnCPU=>operation: memcpy each pointer to end of PoolB
freeMemPool=>operation: Release resources of PoolA
copyOnGPU=>operation: launch kernel to copy values to end of PoolB

st->shouldTakeFromHost
shouldTakeFromHost(yes)->obHostViewFromPoolA->copyOnCPU->freeMemPool
shouldTakeFromHost(no)->obDevViewFromPoolA->copyOnGPU->freeMemPool
```



