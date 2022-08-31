# With CUDA

**Setup GPU cluster & initialize the distributed process group**

```python
def setup(rank, world_size):
    
    os.environ['MASTER_ADDR'] = 'localhost'
    os.environ['MASTER_PORT'] = '12369'

    # initialize the process group
    dist.init_process_group("gloo", rank=rank, world_size=world_size)
```

**Start training by spawning a parallel process on each GPU**

```python
if __name__ == "__main__":
    
    n_gpus = torch.cuda.device_count()
    assert n_gpus >= 2, f"Requires at least 2 GPUs to run, but got {n_gpus}"
    world_size = n_gpus
    
    mp.spawn(main,
             args=(world_size,),
             nprocs=world_size,
             join=True)
         
```
