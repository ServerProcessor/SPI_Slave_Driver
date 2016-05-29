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
	<li><a href="#am335x-and-mcspi-data-sheets">AM335x and McSPI data sheets</a></li>
	<li><a href="#books-about-linux-and-lkm">Books about Linux and LKM</a></li>
	<li><a href="#building-on-beaglebone-board">Building driver on BeagleBone Board</a></li>	
	<li><a href="#building-on-x86-platform">Building driver on x86 platform</a></li>
	<li><a href="#device-tree-overlays">Device Tree Overlays</a></li>
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
<img src="https://raw.githubusercontent.com/pmezydlo/SPI_slave_driver_implementation/master/documentation/block_diagram.png" alt="Block Diagram" width="1114" height="793"/>
	
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
	<ul><li>skeleton LKM ✔</li>
	<li>create debugging tool (dynamic debug) ✔</li>
	<li>prepare test application(using spidev) ✔</li>
	<li>preapre DTS for test application ✔</li>
	<li>create DTS for SPI Slave Device settings ✔</li></ul>
</li>
<li>
	<p>Week 2(31 May - 07 June) </p>
	<ul><li>beginning work on the driver</li>
	<li>create function: probe, remove, init exit</li>
	<li>implementation platform device</li></ul>
</li>
<li>
	<p>Week 3(07 June - 14 June)</p>
	<ul><li>determinate base address (reading DTS device resources)</li>
	<li>create spi slave structure</li>
	<li>allocate memory for device</li>
	<li>fill structures</li>
</ul>
</li>
<li>
	<p>Week 4(14 June - 21 June)</p>
	<ul>
	<li>define McSPI register</li>
	<li>reading and changing registers</li>	
	</ul>
</li>
<li>
	<p>Week 5(21 June - 28 June)</p>
	<ul><li>set McSPI registers in slave mode</li>
	<li>create interrupt(end of transmission)</li>
	</ul>
</li>
<li>
	<p>Week 6(28 June - 05 July)</p>
	<ul>
	<li>allocate memory for tx and rx buffer</li>
	<li>read and write McSPI buffer(PIO)</li>
	<li>first test SPI slave</li></ul>
</li>
<li>
	<p>Week 7(05 July - 12 July)</p>
	<ul><li>start working on the dma</li>
	<li>create DMA structure</li>
	<li>create DMA channel</li></ul>
</li>
<li>
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
		
<h2 id="building-on-beaglebone-board">Building on BeagleBone Board</h2>	
<ol>
	<li>Installing compiler:</li>
	<pre>apt-get install gcc</pre>
	<li>Installing kernel headers:</li>
	<pre>sudo apt-get install linux-headers-'uname -r'</pre>
	<li>Cloning SPI slave repository:</li>
	<pre>git clone git@github.com:pmezydlo/SPI_slave_driver_implementation.git</pre>
	<pre>cd SPI_slave_driver_implementation/</pre>	
	<pre>git checkout</pre>
	<li>Building SPI slave driver:</li>
	<pre>make</pre>				
</ol>
<h2 id="building-on-x86-platform">Building on x86 platform</h2>	
<ol>
	<li>Installing the arm32 cross-compiler:</li>
	<pre>sudo apt-get update -qq</pre>
	<pre>sudo apt-get install -y gcc-arm-linux-gnueabihf</pre>
	<pre>sudo apt-get install bc </pre>
	<li>Cloning SPI slave repository:</li>
	<pre>git clone git@github.com:pmezydlo/SPI_slave_driver_implementation.git</pre>	
	<pre>cd SPI_slave_driver_implementation/</pre>
	<pre>git checkout</pre>	
	<li>Downloading Linux headers:</li>
	<pre>wget https://github.com/beagleboard/linux/archive/4.4.8-ti-rt-r22.tar.gz</pre>
	<li>Unpacking Linux headers:</li>
	<pre>tar zvxf 4.4.8-ti-rt-r22.tar.gz</pre>
	<li>Installing Linux headers:</li>
	<pre>make -j3 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- mrproper</pre>
	<pre>make -j3 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- defconfig</pre>
	<pre>make -j3 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- modules</pre>	
	<pre>cd ../</pre>
	<li>Building SPI slave driver:</li>
	<pre>make</pre>				
</ol>

<h2 id="device-tree-overlays">Device Tree Overlays</h2>	
<p>DTS is required to operate SPI. In repository there is three device tree source which is located in /DTS/ directory.</p>
<h3>Building:</h3>
<ol>
	<li>Go to the directory:</li>
	<pre>cd DTS/</pre>
	<li>Building DTS:</li>
	<pre>dtc -O dtb -o SPI0_master-00A0.dtbo -b 0 -@ SPI0_master.dts</pre>	
	<li>Copying dtbo files to /lib/firmware/:</li>
	<pre>cp SPI0_master-00A0.dtbo /lib/firmware/</pre>
</ol>

<h3>Installing:</h3>
<ol>
	<li>Installing DTS:</li>
	<pre>echo SPI0_master>/sys/devices/platform/bone_capemgr/slots</pre>
	<li>Checking:</li>
	<pre>cat /sys/devices/platform/bone_capemgr/slots</pre>	
	<li>Result:</li>
	<pre> 0: PF----  -1
 1: PF----  -1
 2: PF----  -1
 3: PF----  -1
 5: P-O-L-   0 Override Board Name,00A0,Override Manuf,SPI0_master
</pre>
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