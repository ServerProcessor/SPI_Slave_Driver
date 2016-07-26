<html>
<head>
    <meta charset="UTF-8">
</head>
<body>
<a href="https://summerofcode.withgoogle.com"><img src="https://raw.githubusercontent.com/pmezydlo/SPI_slave_driver_implementation/master/documentation/gsoc-basic-logo.png" alt="GSOC" width="231" height="125"/></a>
<a href="https://beagleboard.org"><img src="https://raw.githubusercontent.com/pmezydlo/SPI_slave_driver_implementation/master/documentation/beagleboard_logo.png" alt="BeagleBoard.org" width="98" height="125"/></a>
<a href="http://www.elinux.org"><img src="https://raw.githubusercontent.com/pmezydlo/SPI_slave_driver_implementation/master/documentation/logo-linux.png" alt="eLinux.org" width="107" height="125"/></a>
	
<h1 id="top">SPI slave driver implementation</h1>
<ol>
	<li><a href="#introduction">Introduction</a></li>
	<li><a href="#links">Links</a></li>
    	<li><a href="#block-diagram">Block diagram</a></li>	
	<li><a href="#slave-driver-in-linux-architecture">Slave driver in Linux architecture</a></li>	
	<li><a href="#problems">Problems</a></li>
	<li><a href="#development-timeline">Development timeline</a></li>
        <li><a href="#photos-from-tests">Photos from tests</a></li>
	<li><a href="#am335x-and-mcspi-data-sheets">AM335x and McSPI data sheets</a></li>
	<li><a href="#books-about-linux-and-lkm">Books about Linux and LKM</a></li>
	<li><a href="#building-on-x86-platform">Building driver on x86 platform</a></li>
	<li><a href="#prepare-bb-board">Prepare BB board</a></li>
	<li><a href="#installing-on-bb-platform">Installing driver on BB platform</a></li>
	<li><a href="#first-data-transfer">First data transfer</a></li>
	<li><a href="#testing-and-wiring">Testing and wiring</a></li>
</ol>

<h2 id="introduction">Introduction</h2>
<p>SPI slave driver implementation. The task is to create a driver controlling SPI hardware controller in slave mode,
		and to ensure optimal performance through the use of DMA and interrupt. Creating an easy to implement realization 
		of SPI slave would definitely help the BeagleBone community members to write applications based on SPI much more 
		easily. The first implementation of my protocol driver is going to example of a bidirectional data exchange. 
		This application will provide the BeagleBone community with valuable experience and will be a good example 
		of SPI slave. Hardware limitations make it impossible to perform any realization of the device using SPI slave. 
		Sending away data to the master during one transaction is not possible. One transaction is enough to receive data 
		by slave device. To receive and send back data, two transactions are needed. The data received from master device 
		in a single transaction is transferred to the user after completing the transaction. The user's reply to received 
		data is sent in the next transaction.</p>
<ul>
        <li>Student: Patryk Mezydlo</li>
	<li>Mentors: Michael Welling, Andrew Bradford, Matt Porter</li>
	<li>Organization: <a href="https://beagleboard.org/">BeagleBoard.org</a>  </li>
</ul>

<h2 id="links">Links</h2>
<ul>	
	<li>Code: <a href="https://github.com/pmezydlo/SPI_slave_driver_implementation">https://github.com/pmezydlo/SPI_slave_driver_implementation</a></li>
	<li>Blog: <a href="https://github.com/pmezydlo/SPI_slave_driver_implementation/wiki"> https://github.com/pmezydlo/SPI_slave_driver_implementation/wiki</a></li>
	<li>Wiki: <a href="http://www.elinux.org/Pmezydlo">http://www.elinux.org/Pmezydlo</a></li>
	<li>GSoC site: <a href="https://summerofcode.withgoogle.com/projects/#5652864013697024"> https://summerofcode.withgoogle.com/projects/#5652864013697024</a> </li>
	<li>Video presentation: <a href="https://www.youtube.com/watch?v=yBgMwMcvcKg">https://www.youtube.com/watch?v=yBgMwMcvcKg</a></li>
</ul>

<h2 id="block-diagram">Block Diagram</h2>
<img src="https://raw.githubusercontent.com/pmezydlo/SPI_slave_driver_implementation/master/documentation/how_pio_works.png" alt="Block Diagram" width="1114" height="793"/>
	
<h2 id="slave-driver-in-linux-architecture">Slave driver in Linux architecture</h2>
<p>The framework in Linux should have 3 layers. In this way, there can be low level 
hardware interfaces to SPI hardware like McSPI in slave mode, a middle layer of common 
functions to call by protocol drivers, and then a top protocol driver layer which can be used to implement. 
</p>
<img src="https://raw.githubusercontent.com/pmezydlo/SPI_slave_driver_implementation/master/documentation/3layer.png" alt="3 Layer Diagram" width="450" height="360"/>
	
<h2 id="problems">Problems</h2>		
<h3>The main problems that don't allow us to create a fully universal generic SPI slave driver.</h3>
<ol>
	<li>The master device can begin the transaction at any time, even if the slave device isn't ready to receive data.</li>
	<li>The transfer can be of any length, as standard SPI does not specify the frame’s length. 
		Theoretically, the master device can send so much data that the slave device would stop receiving it because of memory overflow.</li>
	<li>SPI interface is a rather fast interface (10s or 100s of MHz), so with too fast clock the system can’t keep up with 
		generated interrupts. SPI slave controller will hammer the system with interrupts and the system might become unresponsive.</li>
	<li>Even if you manage to pass data to userspace during the transaction, user will have no 
		time to return the response that will be transferred to the master in the same transaction.</li>
	<li>When transaction is short but often repeated, slave device has no time to set the DMA transfer.</li>
	<li>The register with the contents of the data to be sent must be loaded by DMA before the transaction  
	(high state on cs line). There is no way to load a register of broadcasting prior to the transaction, 
	because you do not know the address of the data to come back before the transaction.</li>
</ol>
<h3>Solution:</h3>
<ol>
	<li>The slave device, after installing the driver and DTS, will be ready to receive the data. 
		The user must set the SPI slave device before the master device.</li>
	<li>During the installation of the driver, the user must declare the maximum depth of data buffer. 
		The length is the same for tx and rx. Transfer may be shorter, but not longer than the declared length.</li>
	<li>The maximum speed of the slave mode specified in the documentation is 16Mhz as the maximum speed of SCLK line.</li>
	<li>Holding the response in memory and sending it in the next transaction solves the problem. 
		The time between the transactions is greater than between the reception and transmission during a single transaction.</li>
	<li>Two-stage transactions allow to use DMA for loading and unloading mcspi buffer.</li>
</ol>

	

<h2 id="development-timeline">Development timeline</h2>	
<p>Each week I will devote a few hours to write the documentation.<p>
<ol>
<li>
<p>Before the first week (before 24 May)</p>
<ul><li>create a GitHub repository ✔</li>
<li>create eLinux page ✔</li>
<li>create wiki page ✔</li>
<li>prepare a few sd card with a Firmware Images ✔</li>
<li>installation of necessary software ✔</li>
<li>create .travis.yml script ✔</li>
<li>create README.md ✔</li>
<li>prepare video presentation ✔</li>
<li>add license file ✔</li>
<li>configure vim for kernel coding style ✔</li></ul>
</li>
<li>
	<p>Week 1(24 May - 31 May)</p>
	<ul><li>create basic function: probe, remove, init exit ✔</li>
<li>prepare test application(using spidev) ✔</li>
<li>add more information about a driver works (describe main problems) ✔</li>
<li>determinate McSPI base address (reading DTS device resources) ✔</li>
<li>create device structure(keeps all the elements of the controller) ✔</li>
<li>allocate memory for device ✔</li>
<li>define McSPI register ✔</li>
<li>function for reading and writing McSPI registers ✔</li>
	</ul>
</li>

<li>
	<p>Week 2(31 May - 07 June) </p>
	<ul><li>set driver in slave mode ✔</li>
	<li>set the majority of important registers, except for those related to DMA, interrupts and fifo ✔</li>
	<li>familiarizing myself with the basics of McSPI controller ✔</li>
	<li>reading and writing McSPI registers ✔</li>
	<li>synchronization (pm_runtime) ✔</li>
	<li>clean up all kernel coding style issues ✔</li>
	<li>resolve all warnings ✔</li></ul>
</li>
<li>
	<p>Week 3(07 June - 14 June)</p>
	<ul><li>familiarizing myself and setting McSPI transmit register ✔</li>
	<li>first test (echo); send back data which the driver receives from master ✔</li>
	<li>setting interrupt (I had a few problems with this) ✔</li>
	<li>adding function which waits for register bit ✔</li>
	<li>second test (interrupt); generating interrupt after 4 bits and checking how subtracted words count ✔</li>
</ul>
</li>

<li<li>
	<p>Week 4(14 June - 21 June)</p>
	<ul><li>transfer data from the FIFO to the driver buffer (for each word length) in both sides ✔</li>
	<li>finalized working on pio transfer ✔</li>
	<li>using tasklets for pio functions ✔</li>
	<li>a lot of tests (various lengths transfers and various length words(bits per word)) ✔</li>
	<li>beginning work on char driver as a framework for slave driver ✔</li>
	</ul>
</li>
<li>
	<p>Week 5(21 June - 28 June)</p>
	<ul><li>registration and removal of char driver for each platform device ✔</li>
	<li>adding functions: open, read, write, release, etc ✔</li>
	<li>adding device to device list ✔</li>
	<li>create application for user on slave device ✔</li></ul>
</li>
<li>
	<p>Week 6(28 June - 05 July)</p>
	<ul>
	<li>finalized work on char driver ✔</li>
	<li>improved prototype slave application and pio transfer ✔</li>
	<li>added  ioctl ✔</li>
	<li>made thorough reconstruction of the driver ✔</li>
</ul>
</li>
<li>
	<p>Week 7(05 July - 12 July)✔</p>
	<ul>
<li>finalized work on thorough reconstruction of the driver ✔</li>
<li>first prototype test ✔</li>
<li>finalized work on ioctl and poll ✔</li>
<li>finalized work on prototype slave application and pio transfer ✔</li>
<li>familiarizing myself with DMA ✔</li>
<li>searched for bugs and repaired them ✔</li>
<li>upgraded travis script ✔</li>
<li>made a nice diagram ✔</li>

</ul>
</li><li>
<p>Week 8(12 July - 17 July)</p><ul>
	<li>configuring dma channel</li>
	<li>interrupt for dma</li>
	<li>create callback functions</li>	
	<li>finally work on DMA</li>
	<li>tests with DMA without API (printk log)</li></ul>
</li>
<li>
	<p>Week 9(19 July - 26 July)</p><ul>
	<li>begining work on the framework</li>
	<li>minor number for charter device</li>
	<li>create class and device</li> 
	<li>create IOCTL command</li>
	<li>structure for sending and receiving data using IOCTL</li></ul>
</li>
<li>
	<p>Week 10(26 July - 02 August)</p><ul>
	<li>create mutex for writing and reading</li>
	<li>asynchronous events for realization flag of notifications</li>
	<li>finally write framework</li></ul>
</li>
<li>
<p>Week 11(02 August - 09 August)</p>
<ul><li>documentation, tutorials and examples for SPI Slave framework</li>
<li>create a make script to compile and install in linux</li> 
<li>more tests</li></ul>
</li>
<li>
<p>Week 12(09 August - 20 August)</p>
<ul><li>clean up code</li>
<li>latest tests</li> 
<li>documentation, tutorials and examples</li></ul>
</li>

</ol>

<h2 id="photos-from-tests">Photos from tests</h2>	
<img src="https://raw.githubusercontent.com/pmezydlo/SPI_slave_driver_implementation/master/documentation/pio_2MHz_img.png" alt="PIO test 2MHz" width="700" height="450"/>	
<img src="https://raw.githubusercontent.com/pmezydlo/SPI_slave_driver_implementation/master/documentation/pio_10MHz_img.png" alt="PIO test 10MHz" width="700" height="450"/>	
<img src="https://raw.githubusercontent.com/pmezydlo/SPI_slave_driver_implementation/master/documentation/pio_16MHz_img.png" alt="PIO test 16MHz" width="700" height="450"/>	
<img src="https://raw.githubusercontent.com/pmezydlo/SPI_slave_driver_implementation/master/documentation/pio_25MHz_img.png" alt="PIO test 20MHz" width="700" height="450"/>	


<h2 id="am335x-and-mcspi-data-sheets">AM335x and McSPI data sheets</h2>	
<ul>
<li>AM335x ARM® Cortex™-A8 Microprocessors (MPUs) Technical Reference Manual:</li>
<a href="http://phytec.com/wiki/images/7/72/AM335x_techincal_reference_manual.pdf">AM335x_Techincal_Reference_Manual</a>
<li>Multichannel Serial Port Interface Technical Reference Manual:</li>
<a href="https://raw.githubusercontent.com/pmezydlo/SPI_slave_driver_implementation/master/documentation/McSPI_Technical_Reference_Manual.pdf">McSPI_Technical_Reference_Manual</a>
</ul>
<h2 id="books-about-linux-and-lkm">Books about Linux and LKM</h2>	
<ul>	<li>LINUX DEVICE DRIVERS, Third Edition, Jonathan Corbet, Alessandro Rubini, and Greg Kroah-Hartman pdf is here[1]</li>
<li>Linux Kernel Development, Third Edition, Robert Love pdf is here[2]</li>
</ul>

<a href="https://lwn.net/Kernel/LDD3/">[1]https://lwn.net/Kernel/LDD3/</a>

<a href="http://infoman.teikav.edu.gr/~stpapad/linux_kernel_development_3rd_edition.pdf">[2]http://infoman.teikav.edu.gr/~stpapad/linux_kernel_development_3rd_edition.pdf</a>		
		

<h2 id="building-on-x86-platform">Building on x86 platform</h2>	
<ol>
<li>Update your linux:</li>
<pre>$ sudo apt-get update -qq</pre>
<pre>$ sudo apt-get install bc </pre>
<li>Cross-compiler toolchain - To download the latest version of cross-compiler tool-chain:</li>
<pre>$ wget -c https://releases.linaro.org/components/toolchain/binaries/5.3-2016.02/arm-linux-gnueabihf/gcc-linaro-5.3-2016.02-x86_64_arm-linux-gnueabihf.tar.xz</pre>
<pre>$ tar xf gcc-linaro-5.3-2016.02-x86_64_arm-linux-gnueabihf.tar.xz</pre>
<pre>$ export CC=`pwd`/gcc-linaro-5.3-2016.02-x86_64_arm-linux-gnueabihf/bin/arm-linux-gnueabihf-</pre>
<li>Cloning SPI slave repository:</li>
<pre>$ git clone git@github.com:pmezydlo/SPI_slave_driver_implementation.git</pre>	
<pre>$ cd SPI_slave_driver_implementation/</pre>
<pre>$ git checkout</pre>	
<li>Check kernel version on your BB board (connect with your board):</li>
<pre>$ ssh root@192.168.7.2</pre>
<pre>$ uname -r</pre>
<pre>4.4.8-ti-r22</pre>
<li>BBB clone kernel source:</li>
<pre>$ export SOURCE_BRANCH="4.4.8"</pre>
<pre>$ export SOURCE_VERSION="ti-r22"</pre>
<pre>$ export SOURCE_REPO="linux-stable-rcn-ee"</pre>
<pre>$ export SOURCE_LOCATION="https://github.com/RobertCNelson"</pre>
<pre>$ wget "$SOURCE_LOCATION/$SOURCE_REPO/archive/$SOURCE_BRANCH-$SOURCE_VERSION.tar.gz"</pre>
<pre>$ tar xf $SOURCE_BRANCH-$SOURCE_VERSION.tar.gz</pre>
<pre>$ export DST_KERNEL=$PWD/$SOURCE_REPO-$SOURCE_BRANCH-$SOURCE_VERSION</pre>	
<li>Cross compiling the kernel source:</li>
<pre>$ cd $DST_KERNEL</pre>
<pre>$ make -j3 mrproper ARCH=arm CROSS_COMPILE=$(CC) LOCALVERSION=-$SOURCE_VERSION</pre>
<pre>$ wget -c "http://rcn-ee.net/deb/jessie-armhf/v$SOURCE_BRANCH-$SOURCE_VERSION/defconfig" -O .config</pre>
<pre>$ make -j3 modules ARCH=arm CROSS_COMPILE=$(CC) LOCALVERSION=-$SOURCE_VERSION 2>&1</pre> 
<li>Cross compile SPI slave driver:</li>
<pre>make KDIR=$DST_KERNEL ARCH=arm CROSS_COMPILE=$(CC) LOCALVERSION=-$SOURCE_VERSION</pre>	
<li>Check the driver version:</li>
<pre>$ modinfo driver/spi-mcspi-slave.ko</pre>
<pre>filename:       /home/pmezydlo/BeagleBoard_work/SPI_slave_driver_implementation/driver/spi-mcspi-slave.ko</pre> 
<pre>version:        1.0</pre> 
<pre>description:    SPI slave for McSPI controller.</pre> 
<pre>author:         Patryk Mezydlo, <mezydlo.p@gmail.com></pre> 
<pre>license:        GPL v2</pre> 
<pre>srcversion:     469EA334B14612EBD6F0463</pre> 
<pre>alias:          of:N*T*Cti,omap4-mcspi*</pre> 
<pre>depends:        </pre> 
<pre>vermagic:       4.4.8-ti-r22 SMP mod_unload modversions ARMv7 thumb2 p2v8 </pre> 
</ol>

<h2 id="prepare-bb-board">Prepare BB board</h2>	

<ol><h3>Disabling the BeagleBone black hdmi cape</h3>
<li>Mount the FAT partition:</li>
<pre>$ mount /dev/mmcblk0p1  /mnt/card</pre> 
<li>Edit the uEnv.txt on the mounted partition:</li>
<pre>$ nano /mnt/card/uEnv.txt</pre> 
<li>To disable the HDMI Cape, change the contents of uEnv.txt to:</li>
<pre>$ optargs=quiet capemgr.disable_partno=BB-BONELT-HDMI,BB-BONELT-HDMIN</pre> 
<li>Save the file:</li>
<pre>Ctrl-X, Y</pre> 
<li>Unmount the partition:</li>
<pre>$ umount /mnt/card</pre> 
<li>Reboot the board:</li>
<pre>$ shutdown -r now</pre> 
<li>Wait about 10 seconds and reconnect to the BeagleBone Black through SSH. To see what capes are enabled:</li>
<pre>$ cat /sys/devices/bone_capemgr.*/slots</pre> 
</ol>


<ol><h3>Disabling the spi master driver(spi_omap2_mcspi)</h3>
	
<li>Create a file:</li>
<pre>$ nano /etc/modprobe.d/spi_omap2_mcspi.conf</pre> 
<li>Edit the spi_omap2_mcspi.conf:</li>
<pre>blacklist spi_omap2_mcspi</pre> 
<li>Save the file:</li>
<pre>Ctrl-X, Y</pre> 
<li>Reboot the board:</li>
<pre>$ shutdown -r now</pre> 
</ol>

<ol><h3>Installing the Device Tree Overlays(DTS)</h3>
	
<li>Installing the DTS compiler:</li>
<pre>$ wget -c https://raw.githubusercontent.com/RobertCNelson/tools/master/pkgs/dtc.sh</pre>
<pre>$ chmod +x dtc.sh</pre>
<pre>$ ./dtc.sh</pre>
<li>Compiling the SPI slave dts file:</li>
<pre>$ cd DTS/</pre>
<pre>dtc -O dtb -o SPI0_slave-00A0.dtbo -b 0 -@ SPI0_slave.dts</pre>
</ol>

<ol><h3>Upload spi slave driver, dtb and slave application on BB board:</h3>
	
<li>Upload using SCP:</li>	
<pre>$ scp driver/spi-mcspi-slave.ko  root@192.168.137.2:/root</pre>
<pre>$ scp slave_app/slave_app  root@192.168.137.2:/root</pre>
<pre>$ scp DTS/SPI0_slave-00A0.dtbo  root@192.168.137.2:/lib/firmware</pre>
</ol>


<h2 id="installing-on-bb-platform">Installing on BB platform</h2>	

<ol><h3>Installing DTS:</h3>
<li>Make an entry:</li>
<pre>echo SPI0_slave>/sys/devices/platform/bone_capemgr/slots</pre>
<li>Checking:</li>
<pre>$ cat /sys/devices/platform/bone_capemgr/slots</pre>	
<pre> 0: PF----  -1</pre>
<pre> 1: PF----  -1</pre>
<pre> 2: PF----  -1</pre>
<pre> 3: PF----  -1</pre>
<pre> 5: P-O-L-   0 Override Board Name,00A0,Override Manuf,SPI0_slave</pre>
</ol>


<ol><h3>Installing driver:</h3>
<li>Command insmod:</li>
<pre>insmod spi-mcspi-slave.ko </pre>
<li>Checking:</li>
<pre>$ lsmod</pre>	
<pre>Module                  Size  Used by</pre>
<pre>spi_mcspi_slave        10781  0 </pre></ol>


<h2 id="first-data-transfer">First data transfer</h2>	
<ol>
</ol>

<h2 id="testing-and-wiring">Testing and wiring</h2>	
<h3>The first option:</h3>
<p>One AM335x processor contains two McSPI controllers, 
what allows to use one controller as  slave and the 
other as  master. This allows to carry out tests on one board.</p>
<img src="https://raw.githubusercontent.com/pmezydlo/SPI_slave_driver_implementation/master/documentation/one_bb.png" alt="one beaglebone" width="242" height="338"/>	
<h3>The second option:</h3>
<p>The second option is to use two boards where one works as master and the other as slave.</p>
<img src="https://raw.githubusercontent.com/pmezydlo/SPI_slave_driver_implementation/master/documentation/two_bb.png" alt="two beaglebones" width="436" height="338"/>
</body>
</html>