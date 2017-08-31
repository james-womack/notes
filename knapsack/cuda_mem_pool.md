## Plans for memory pool

**Create a class that will hold a pool of memory representing a subset of the set of possible promising solutions so far.**

- Making device pointers to the raw set of memory, and with pointers pointing to offsets in the data to the different sections. Same for the host side.
- Keep track of the current state of the pool and facilitate the ability to migrate data from one pool to another.
    - Make a function that merges two memory pools together in order to reduce memory footprint:

        ```c++
        std::shared_ptr<gpu_mem_pool> merge(std::shared_ptr<gpu_mem_pool>& a, std::shared_ptr<gpu_mem_pool>& b);
        ```

    - Make a function that splits a memory pool into two when a threshold is hit:

        ```c++
        std::shared_ptr<gpu_mem_pool> split(std::shared_ptr<gpu_mem_pool>& a);
        ```

- Create a struct of which represents a view into the memory pool for a certain node, this will be useful for the host for conciseness of the algorithm and can be used on the GPU side if desired.
    - Create a simple struct that represents a possible solution:

        ```c++
        struct bfs_solution_view {
           int* m_beforeSlackValue;
           int* m_beforeSlackWeight;
           int* m_slackItem;
           int* m_ubValue;
           int* m_lbValue;
           int* m_label;
           bool* solutionVector;
        };
        ```

    - Create a function that gets a view into the memory pool, made for iterating over the memory pool:

        ```c++
        void gpu_mem_pool::get_node(const int& pId, bfs_solution_view& view);
        ```

- Modify existing algorithm to take advantage of the memory pools.
      - Modify all of the functions in **gpu_alg.cu[h]** **gpu_alg_gpu.cu[h]** to use these memory pools
      - Make them synchronize the memory pools when we find the best lower bound, concatenate lists, etc...
            - This means iterating over all of the memory pools and applying all of the kernels to each one of them, investigate usage into *async memcpys* and *async scheduling*.
      - Move CPU version into host memory allocated and use **bfs_solution_view** strict in order to represent each solution, as a view into the memory pool. This should allow us to match GPU algorithm closer and make it easily parallizable. ​