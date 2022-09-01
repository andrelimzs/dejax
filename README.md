# dejax
An implementation of replay buffer data structure in JAX. Operations involving dejax replay buffers can be jitted and run on both CPU and GPU.

## Package contents
* `dejax.circular_buffer` — an implementation of a circular buffer data structure that can store pytrees of arbitrary structure (with the restriction that the corresponding tensor shapes in different pytrees match).
* `dejax.uniform` — a FIFO replay buffer of fixed size that samples replayed items uniformly.
* `dejax.clustered` — a replay buffer that assigns every trajectory to a cluster and maintains a separate replay buffer per cluster. Sampling is performed uniformly over all clusters. This kind of replay buffer is helpful when, for instance, one needs to replay low and high reward trajectories at the same rate.

## How to use dejax replay buffers

```python
import dejax
```

First, instantiate a buffer object. Buffer objects don't have state but rather provide methods to initialize and manipulate state.

```python
buffer = uniform_replay(max_size=10000)
```

Having a buffer object, we can initialize the state of the replay buffer. For that we would need a prototype item that will be used to determine the structure of the storage. The prototype item must have the same structure and tensor shapes as the items that will be stored in the buffer.

```python
buffer_state = buffer.init_fn(item_prototype)
```

Now we can fill the buffer:
```python
for item in items:
    buffer_state = buffer.add_fn(buffer_state, make_item(item))
```

And sample from it:
```python
batch = buffer.sample_fn(buffer_state, rng, batch_size)
```

Or apply an update op to the items in the buffer:
```python
def item_update_fn(item):
    # Possibly update an item
    return item
buffer_state = buffer.update_fn(buffer_state, item_update_fn)
```

### An important note regarding in-place buffer updates
JAX currently does not allow to update arguments of jit-compiled functions in-place even if the arguments are donated. It means that the following code
```python
train_state, replay_buffer_state = jax.jit(train_step, donate_argnums=(1,))(
    train_state, replay_buffer_state, trajectory_batch
)
```
will produce a copy of the replay buffer state if something has been added to the buffer inside `train_step`. This may cause the performance benefits of having access to a replay buffer in jit-compiled code to vanish. A relevant discussion of this issue can be found [here](https://github.com/google/jax/issues/9132#issuecomment-1234332780). Hopefully it will be fixed in future versions of JAX.
