<html>
<head>
    <meta charset="UTF-8">
</head>
<body>

<h1 id="top">SPI slave driver implementation</h1>
<ol>
	<li><a href="#intro">Introduction</a></li>
	<li><a href="#links">Link</a></li>
	<li><a href="#buildBBB">Building driver on BeagleBone Board</a></li>	
        <li><a href="#buildx86">Building driver on x86 platform</a></li>
</ol>

<h2 id="intro">SPI slave driver implementation</h2>
<p>The project enables easy to implement realization of SPI slave. 
Driver contains basic functions to communicate with SPI controller(McSPI) 
and API for communicate with userspace and to ensure optimal performance 
through the use of DMA and interrupt.</p>
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
		
		
<h2 id="buildBBB">Building on BeagleBone Board</h2>	
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
<h2 id="buildx86">Building on x86 platform</h2>	
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

</body>
</html>