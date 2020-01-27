The recommended way of generating JSON profiles is now using Volatility3/dwarf2json, but in case anyone still nedds Rekall, the instructions are as follows.

Rekall is unfortunately buggy with newer versions of Python3 so we have to stick to Python 3.5.7:

<pre><code>wget https://www.python.org/ftp/python/3.5.7/Python-3.5.7.tar.xz
tar xvf Python-3.5.7.tar.xz
cd Python-3.5.7
./configure --enable-optimizations --prefix=/opt/python3.5.7
sudo make altinstall
cd ..
</code></pre>

Now lets make Python 3 the default and install some required packages:

<pre><code>sudo update-alternatives --install /usr/bin/python python /opt/python3.5.7/bin/python3.5 1
sudo /opt/python3.5.7/bin/pip3.5 install fastchunking wheel future==0.16.0
</code></pre>

Install Rekall from git:

<pre><code>git clone --depth=1 https://github.com/tklengyel/rekall
cd rekall/rekall-core
./setup.py build
sudo ./setup.py install
</code></pre>

If vmi-win-guid fails to find the Windows kernel in memory, you can use Rekall to examine ntoskrnl.exe on the disk:
<pre><code>sudo su
kpartx -a /dev/vg/windows7
mount -o ro /dev/vg/windows7p1 /mnt
python3 rekal peinfo -f /mnt/Windows/System32/ntoskrnl.exe > /tmp/peinfo.txt
umount /mnt
kpartx -d /dev/vg/windows7
</code></pre>

The generated /tmp/peinfo.txt file will contain the required PDB filename and GUID.

Now generate the Rekall profile (make sure to adjust the kernel name and GUID as necessary):
<pre><code>cd /tmp
python3 rekall fetch_pdb ntkrpamp 684da42a30cc450f81c535b4d18944b12
python3 rekall parse_pdb ntkrpamp > windows7-sp1.rekall.json
</code></pre>

For Linux you need to build the initial kernel profile in the guest itself.
<pre><code>ssh root@linux
apt-get install git zip linux-headers-$(uname -r) build-essential
git clone --depth=1 https://github.com/tklengyel/rekall
cd rekall/tools/linux
make
</code></pre>

This will generate a ZIP file with your kernel-version as filename. For example, 3.16.0-4-amd64.zip. Copy this file to your DRAKVUF&trade; host (for example using scp). There we will conver$
<pre><code>rekal convert_profile 3.16.0-4-amd64.zip /root/linux.json
</code></pre>
