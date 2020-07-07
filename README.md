### Benchmarking method

Each tested EC2 instance is provisioned with an [1280GiB io1][1] volume.
The size is chosen to allow the highest possible IOPS guarantee for a
single EBS: `64000 IOPS`.

The following lines are executed on every individual EC2, including a
kernel-upgrading reboot. Once the provisional git branch is assembled, it is
pulled into this repository and made available [for a github diff][2].

* NOTE: when diffing on github you need to change the `...` to a `..` in
the `URL` for the diff rendering to make sense.


```
sudo apt-get update ; sudo apt-get upgrade -y ; sudo apt-get install -y fio dstat git numactl curl
sudo apt-get remove --purge linux-{image,modules,headers}-5.4.0-1015-aws

sudo mkfs.xfs -L spaaace \
  -s size=4k -b size=4k \
  -n version=2,ftype=1,size=8k \
  -m crc=1,finobt=1,rmapbt=1 \
  -i size=512,maxpct=20,align=1,attr=2,projid32bit=1,sparse=1 \
  -l internal=1,version=2,lazy-count=1 \
  /dev/nvme1n1

sudo update-initramfs -k all -u ; sudo update-grub ; sudo reboot

sudo mount LABEL=spaaace /srv

git init benchres
cd benchres
dpkg -l > ubuntu-packages.txt
curl -s http://169.254.169.254/latest/meta-data/instance-type > aws-machine-type.txt
curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone > aws-region-az.txt
uname -a > uname.txt
lscpu > cpu.txt
numactl -H > mem-arch.txt
openssl speed sha > sha-bench.txt
for jobs in 1 4 8 ; do \
  for engine in libaio psync mmap ; do \
    echo 3 | sudo tee /proc/sys/vm/drop_caches > /dev/null
    sudo fio --name="$( cat aws-machine-type.txt )" --rw=randrw --size=128M --iodepth=4 --direct=1 --directory=/srv --numjobs=$jobs --ioengine=$engine --output=fio_${engine}_${jobs}-threads.txt
  done
done

(
  echo -e '!!! Note: pricing DOES NOT reflect TCO due to the wild and widely differing AWS margin !!!\n'
  curl -s "https://ec2.shop?filter=$( cat aws-machine-type.txt )"
) > aws-price-quote-retail.txt

git checkout -b "iobench_$( cat aws-machine-type.txt )_$( TZ=UTC date +%Y-%m-%d_%H-%M-%S )"
git add .
git commit -m "AWS benchmark results"
```

[1]: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-volume-types.html#EBSVolumeTypes_piops
[2]: https://github.com/ribasushi/basic-aws-io-benchmark/compare?diff=split&expand=1
