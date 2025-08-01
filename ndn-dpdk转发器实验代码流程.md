## ndn-dpdk转发器实验代码流程

### ping.go

Before: openUplink（）

```
l3face := face.Face()
type Face interface {
	Transport() Transport
	Rx() <-chan *ndn.Packet
	Tx() chan<- ndn.L3Packet
	State() TransportState
	OnStateChange(cb func(st TransportState)) (cancel func())
}

fw := l3.GetDefaultForwarder()
	defaultForwarder = NewForwarder()
	func NewForwarder() Forwarder {
	fw := &forwarder{
		faces:         map[uint32]*fwFace{},
		announcements: multimap.NewMapSlice[string, *fwFace](),
		readvertise:   set.NewMapset[ReadvertiseDestination](),
		cmd:           make(chan func()),
		rx:            make(chan fwRxPkt),
	}
	fmt.Println("[forwer]fw.loop() start")
	go fw.loop()
	return fw
}

type fwRxPkt struct {
	*ndn.Packet
	rxFace *fwFace
}

func (fw *forwarder) loop() {
	for {
		select {
		case fn := <-fw.cmd:
			fn()
		case pkt := <-fw.rx:
			switch {
			case pkt.Interest != nil:
				fw.forwardInterest(pkt)
			case pkt.MyInterest != nil:
				fw.forwardMyInterest(pkt) // 新增处理 MyInterest 的逻辑
				fmt.Println("[forwarder]forwarder receive pkt.MyInterest")
			case pkt.Data != nil, pkt.Nack != nil:
				fw.forwardDataNack(pkt)
			}
		}
	}
}

func (fw *forwarder) forwardInterest(pkt fwRxPkt) {
	lpmLen := 0
	var nexthops []*fwFace
	for _, f := range fw.faces {
		if pkt.rxFace == f {
			continue
		}
		matchLen := f.lpmRoute(pkt.Interest.Name) //待改
		switch {
		case matchLen > lpmLen:
			lpmLen = matchLen
			nexthops = nil
			fallthrough
		case matchLen == lpmLen:
			nexthops = append(nexthops, f)
		}
	}

	for _, f := range nexthops {
		fmt.Println("[forwarder]forwarder send interest to f.rx (forwarder.[]*fwFace.rx (chan<- ndn.L3Packet))")
		f.tx <- pkt
	}
}

func (f *fwFace) lpmRoute(query ndn.Name) int {
	for _, name := range f.routes {
		if name.IsPrefixOf(query) {
			return len(name)
		}
	}
	return -1
}

fwFace, e = fw.AddFace(l3face)
fwFace.AddRoute(ndn.Name{})
fw.AddReadvertiseDestination(face)
```



```
face, e := NewLFace(opts.Fw)
// NewLFace creates a logical face to an internal forwarder.
func NewLFace(fw l3.Forwarder) (face *LFace, e error) {
	if fw == nil {
		fw = l3.GetDefaultForwarder()
	}
	face = &LFace{
		ep2fw: make(chan *ndn.Packet, 16),
		fw2ep: make(chan ndn.L3Packet, 16),
	}
	l3face := lFaceL3{face}
	face.FwFace, e = fw.AddFace(l3face)
	return face, e
}

type LFace struct {
	ep2fw  chan *ndn.Packet
	fw2ep  chan ndn.L3Packet
	FwFace l3.FwFace
}

// Rx returns a channel for receiving packets from internal forwarder.
func (face *LFace) Rx() <-chan ndn.L3Packet {
	return face.fw2ep
}

// Tx returns a channel for sending packets to internal forwarder.
func (face *LFace) Tx() chan<- *ndn.Packet {
	return face.ep2fw
}

// Send attempts to send a packet to internal forwarder.
// Returns true if packet is queued, or false if packet is dropped.
func (face *LFace) Send(pkt *ndn.Packet) bool {
	select {
	case face.ep2fw <- pkt:
		return true
	default:
		return false
	}
}
```



主机A

```
ndndpdk-ctrl create-eth-port --pci 03:00.0 --mtu 1500
true
{"id":"NQLH44GGLLNHS","isDown":false,"macAddr":"00:0c:29:a1:cb:21","mtu":1500,"name":"0000:03:00.0","numaSocket":null}
root@lwj-virtual-machine:/home/lwj/Desktop/ndn-dpdk/mk# ndndpdk-ctrl create-ether-face --local 00:0c:29:a1:cb:21 --remote 00:0c:29:57:30:2a
{"id":"NMV1ICQFT9G1VO9D"}
root@lwj-virtual-machine:/home/lwj/Desktop/ndn-dpdk/mk# ndndpdk-ctrl insert-fib --name /example/P --nh NMV1ICQFT9G1VO9D
{"id":"NMR1G4ORLSJLFRGTG3GLGDNAJNB7AH3VRS"}
root@lwj-virtual-machine:/home/lwj/Desktop/ndn-dpdk/mk# ndndpdk-godemo pingclient --name /example/P
2024/10/21 16:11:40 uplink opened, state is down
2024/10/21 16:11:40 uplink state changes to up
my_interest:  /8=example/8=P/8=E70A521297F7D095
prepare:[consumer]Send my_interest
[fwface]forwarder receive my_interest from *fwFace.Rx() (Rx() <-chan *ndn.Packet)
[forwarder]my_interest name :  /8=example/8=P/8=E70A521297F7D095
[forwarder]forwarder receive pkt.MyInterest
Wire : 642e6204d2b505b350263024071e08076578616d706c650801500810453730413532313239374637443039350c0203e8
[face.txLoop()]send pkt



root@lwj-virtual-machine:/home/lwj/Desktop/ndn-dpdk/mk# ndndpdk-ctrl list-fib
{"id":"NMR1G4ORLSJLFRGTG3GLGDNAJNB7AH3VRS","name":"/8=example/8=P","nexthops":[{"id":"NMV1ICQFT9G1VO9D"}],"strategy":{"id":"L2LGGDO1NOP5FRH4"}}
```

主机B

```
ndndpdk-ctrl activate-forwarder < /home/lwj/Desktop/forwarder.schema.json
ndndpdk-ctrl create-eth-port --pci 03:00.0 --mtu 1500
true
{"id":"T7PHMSNLA6CPK","isDown":false,"macAddr":"00:0c:29:57:30:2a","mtu":1500,"name":"0000:03:00.0","numaSocket":null}
root@lwj-virtual-machine:/home/lwj/Desktop# ndndpdk-ctrl create-ether-face --local 00:0c:29:57:30:2a --remote 00:0c:29:a1:cb:21
{"id":"TBJ10KTA2MD940K8"}
root@lwj-virtual-machine:/home/lwj/Desktop# ndndpdk-godemo pingserver --name /example/P
[forwer]fw.loop() start
[forwer](fwface)f.rxLoop() start
2024/10/21 16:11:06 uplink opened, state is down
[forwer](fwface)f.rxLoop() start
2024/10/21 16:11:06 uplink state changes to up

root@lwj-virtual-machine:/home/lwj# ndndpdk-ctrl list-fib
{"id":"TBN12SVUAF8T625OE42R629Q7B202MG9R8","name":"/8=example/8=P","nexthops":[{"id":"TBJ10KTA2EB9K141"}],"strategy":{"id":"VVPG2LV48B2D6241"}}

```

