**Question 1**: Solve this question on: ``ssh cks3477``
There are four Kubernetes server binaries located at ``/opt/course/6/binaries`` on ``cks3477``. You're provided with the following verified sha512 values for these:

kube-apiserver
``f417c0555bc0167355589dd1afe23be9bf909bf98312b1025f12015d1b58a1c62c9908c0067a7764fa35efdac7016a9efa8711a44425dd6692906a7c283f032c``

kube-controller-manager
``60100cc725e91fe1a949e1b2d0474237844b5862556e25c2c655a33b0a8225855ec5ee22fa4927e6c46a60d43a7c4403a27268f96fbb726307d1608b44f38a60``

kube-proxy
``52f9d8ad045f8eee1d689619ef8ceef2d86d50c75a6a332653240d7ba5b2a114aca056d9e513984ade24358c9662714973c1960c62a5cb37dd375631c8a614c6``

kubelet
``4be40f2440619e990897cf956c32800dc96c2c983bf64519854a3309fa5aa21827991559f9c44595098e27e6f2ee4d64a3fdec6baba8a177881f20e3ec61e26c``

Delete those binaries that don't match the sha512 values above.

**Step 1 — Switch context and SSH**
```
ssh cks3477
sudo -i
```

**Step 2 — Navigate to the binaries directory**
```
cd /opt/course/6/binaries
ls -la
# Should show: kube-apiserver, kube-controller-manager, kube-proxy, kubelet
```

**Step 3 — Generate sha512 hashes for all binaries**
There are 2 ways you can solve this
```
sha512sum kube-apiserver
```
But the problem with this approach is that you will have to verify all the characters which is time consuming

The other aproach is
```
echo "f417c0555bc0167355589dd1afe23be9bf909bf98312b1025f12015d1b58a1c62c9908c0067a7764fa35efdac7016a9efa8711a44425dd6692906a7c283f032c kube-apiserver" sha512sum --check
# excepted value is "kube-apiserver: OK"
```

Repate the steps for all the files and delete the ones which don't match.