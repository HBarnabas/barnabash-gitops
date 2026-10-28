# PC k3s setup

The PC cluster uses k3s like the Raspberry Pi cluster. k3s supplies the
`local-path` StorageClass and ServiceLB, so no additional storage provisioner
or load balancer is needed.

For the shared HDD-backed volumes, mount the PC's data disk at `/mnt/hdd`
before starting workloads. The PVs use host paths under that mount, matching
the Raspberry Pi setup. Adjust those paths in the shared app manifests if the
PC uses a different mount point.
