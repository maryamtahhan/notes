# xdp-hints

The following diagram shows the call sequence to pass xdp-hints from the ixgbe-driver to an application using AF_XDP. The main idea is that xdp-hints are stored in the xdp-headroom area of an xdp-buff. The flags for the buffer are then updated to indicate that there are hints available for consumption. When an application consumes an xdp-buff, they can check the flags and then process the information in the buffers headroom.

While there's no restriction on the type of information that can be passed through xdp-hints. A key requirement is that the 64 bytes preceding the data in an xdp-buff is reserved for the BTF id that identifies the information aka the struct type of the xdp-hint.

![pass-hints](http://www.plantuml.com/plantuml/proxy?cache=no&src=https://raw.githubusercontent.com/maryamtahhan/notes/main/xdp-hints/plantuml/rx-path.puml)

![napi-pass-hints](http://www.plantuml.com/plantuml/proxy?cache=no&src=https://raw.githubusercontent.com/maryamtahhan/notes/main/xdp-hints/plantuml/rx-napi-path.puml)

> **_NOTE:_** xskq_prod_reserve_desc() will enqueue a descriptor to the AF_XDP
Receive Ring - which will later be consumed by CNDPs xskdev API.

## struct xdp_buff and xdp-hints

The hints and the flags are currently being set by the driver.

```c
struct xdp_buff {
    void *data;
    void *data_end;
    void *data_meta;
    void *data_hard_start;
    struct xdp_rxq_info *rxq;
    struct xdp_txq_info *txq;
    u32 frame_sz; /* frame size to deduce data_hard_end/reserved tailroom*/
    u32 flags; /* supported values defined in xdp_buff_flags */
};
```

![xdp-buff](./images/xdp-buff.png)

The xdp_buff struct is passed all the way from the driver ixgbe_clean_rx_irq() function to __xsk_rcv_zc(). In the case of AF_XDP , when a buffer reaches __xsk_rcv_zc() the flag information contained in xdp_buff
is converted/passed to an xdp descriptor that gets added to the AF_XDP receive
ring. The idea is to reuse the options field in struct xdp_desc to pass the
hints flags to CNDP.

> **_NOTE:_** User space applications consuming AF_XDP buffers simply receive a
> descriptor that gives them:
>
>1. An offset into the memory area (UMEM) where the packet data is.
>2. The length of that data
>3. An options field that leverages flags to indicate different information to the
> application (like if xdp-hints are available in the buffer).

```c
/* Rx/Tx descriptor */
struct xdp_desc {
    __u64 addr;
    __u32 len;
    __u32 options;
};
```

## xdp_do_redirect ()

xdp_do_redirect() selects the redirection function based on the map type

```c
int xdp_do_redirect(struct net_device *dev, struct xdp_buff *xdp,
            struct bpf_prog *xdp_prog)
{
    struct bpf_redirect_info *ri = this_cpu_ptr(&bpf_redirect_info);
    enum bpf_map_type map_type = ri->map_type;

    /* XDP_REDIRECT is not fully supported yet for xdp frags since
     * not all XDP capable drivers can map non-linear xdp_frame in
     * ndo_xdp_xmit.
     */
    if (unlikely(xdp_buff_has_frags(xdp) &&
             map_type != BPF_MAP_TYPE_CPUMAP))
        return -EOPNOTSUPP;

    if (map_type == BPF_MAP_TYPE_XSKMAP)
        return __xdp_do_redirect_xsk(ri, dev, xdp, xdp_prog);

    return __xdp_do_redirect_frame(ri, dev, xdp_convert_buff_to_frame(xdp),
                       xdp_prog);
}
```

> **_NOTE:_** struct xdp_buff *xdp is passed all the way through to __xsk_rcv_zc()
where it's then replaced with an xsk descriptor using the xskq_prod_reserve_desc()
function.

## __xsk_rcv_zc ()

The information container in struct xdp_buff *xdp is used to configure a descriptor
of type struct xdp_desc. This information in configured in the skq_prod_reserve_desc() function. It's important to note that the xdp-buff flags need to be translated to AF_XDP options and sent as part of the descriptor so an end application knows that there are xdp-hints available in this buffer. This is done with the xdp_buff_has_hints_compat() and xdp_buff_has_hints() checks in __xsk_rcv_zc().

```c
static int __xsk_rcv_zc(struct xdp_sock *xs, struct xdp_buff *xdp, u32 len)
{
    struct xdp_buff_xsk *xskb = container_of(xdp, struct xdp_buff_xsk, xdp);
    u64 addr;
    int err;
    u32 flags = 0;

    addr = xp_get_handle(xskb);
    if (xdp_buff_has_hints_compat(xdp))
        flags |= XSK_DESC_HAS_HINTS_COMMON;
    if (xdp_buff_has_hints(xdp))
        flags |= XSK_DESC_HAS_HINTS;

    err = xskq_prod_reserve_desc(xs->rx, addr, len, flags);
    if (err) {
        xs->rx_queue_full++;
        return err;
    }

    xp_release(xskb);
    return 0;
}
```

## xskq_prod_reserve_desc()

```c
static inline int xskq_prod_reserve_desc(struct xsk_queue *q,
                     u64 addr, u32 len, u32 options)
{
    struct xdp_rxtx_ring *ring = (struct xdp_rxtx_ring *)q->ring;
    u32 idx;

    if (xskq_prod_is_full(q))
        return -ENOBUFS;

    /* A, matches D */
    idx = q->cached_prod++ & q->ring_mask;
    ring->desc[idx].addr = addr;
    ring->desc[idx].len = len;
    if (options)
        ring->desc[idx].options = options;

    return 0;
}
```

## Application receiving AF_XDP buffers

An application receiving a descriptor that's pointing to an xdp-buff in the umem
needs to check if there's hints set on the incoming descriptor options. An example from CNDP is shown below:

```c
static __cne_always_inline uint16_t
__get_mbuf_rx_aligned(void *_xi, void *umem_addr, const struct xdp_desc *d, void **bufs)
{
    xskdev_info_t *xi = _xi;
    void *addr;
    uint64_t offset, mask = (xi->buf_mgmt.frame_size - 1);

    addr   = (void *)d->addr; /* Get the offset to the buffer in umem */
    offset = (uint16_t)((uint64_t)addr & mask);
    addr   = CNE_PTR_SUB(addr, offset);

    /* Replace addr with the pointer to the umem packet data in umem */
    *bufs = xsk_umem__get_data(umem_addr, (uint64_t)addr + xi->buf_mgmt.pool_header_sz);

    xskdev_buf_set_data_len(xi, *bufs, d->len);
    xskdev_buf_set_data(xi, *bufs, offset - xi->buf_mgmt.buf_headroom);

    if (xsk_desc_has_hints(d) || xsk_desc_has_hints_common(d))
        xskdev_buf_process_xdp_hints(xi, *bufs);

    return d->len;
}
```

## AF_XDP Ring overview

![AF_XDP Ring overview](https://raw.githubusercontent.com/CloudNativeDataPlane/cndp/main/doc/guides/prog_guide/img/umem_mbuf.svg)

## References

[xdp-hints LPC 2022](https://lpc.events/event/16/contributions/1362/attachments/1056/2017/xdp-hints-lpc2022.pdf)
