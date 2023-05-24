# Performance-Evaluation-of-Dynamic-Adaptive-Streaming-over-Information-Centric-Networking

*Have packaged the environment and uploaded it to \textbf{Docker Hub}\footnote{docker pull 27718842/dash_ndnsim_samus }.*

### Personal Experimental Notebook
Today, I delved into the world of Information-Centric Networking (ICN) with amust-ndnSIM. The simulation provides a platform to evaluate the performance of Dynamic Adaptive Streaming over ICN.

Within this tutorial, we will try to cover  examples like file-transfers, up to building a large example with several multimedia streaming clients(DASH). We assume that multimedia data(dataset) are stored in /home/someuser/multimediaData, whereas you might want to replace someuser with your username.

This tutorial is organized as follows: Section 1 explains how the file transfers are organized and implemented. Section 2 explains how adaptive multimedia streaming can be used. Last but not least, we will show how to generate a large network using BRITE in Section 3.

Throughout the tutorial we place any code in the ndnSIM/ns-3/scratch folder, and that executing the scenarios by using ./waf --run scenario_name within the ndnSIM/ns-3 folder.

------------------
## 1. File Transfers
The implemented application that come close are the consumer and producer app. 

First, we will talk about the basics of the file transfer, followed by an example of how to host and request the content. Last but not least, we will provide a full scenario and information about tracing.

### Basics
Therefore we implemented a very basic FileServer (resp. ``ns3::ndn::FileServer``) and a FileConsumer (resp. ``ns3::ndn::FileConsumer``) for basic file transfers. Configuring the FileServer is simple: It only needs to know under which prefix (e.g., /myprefix) it needs to host files and where to find files to host. The FileServer automatically handles fragmentation/segmentation of files that do not fit within a single packet. Based on the Maximum Transmission Unit (MTU), the FileServer aims to always fully utilize the MTU by automatically maximizing the payload size. This immediately leads to a limitation of the network: The MTU needs to be the same for the whole network (e.g., 1500).

We achieved fragmentation by providing a so called Manifest for each file, which contains the number of fragments to request and the file size. A typical file transfer based on this implementation looks like this: first, an interest for ``/myprefix/app/File1.txt/Manifest`` is issued. Second, the consumer will start to sequentially issue Interests for the chunks: ``/myprefix/app/File1.txt/1``, ``/myprefix/app/File1.txt/2``, ... ``/myprefix/app/File1.txt/#n`` (where #n is the number of fragments). The numbers (also referred to as sequence numbers) are appeneded to the interest with a binary encoding by using ``ndn::Name::appendSequenceNumber``. 
The FileServer will respond with the respective payload, and the file consumer will stop requesting once it has received the whole file. The consumer also handles timeouts and re-transmissions if necessary. 



## Hosting Content (Producer)
Before we start, we would like to point out that our examples are very similar to the basic examples provided by the [ndnSIM tutorial page](http://ndnsim.net/2.1/examples.html). Please have a look at the examples there before you start working.

Hosting content is straightforward. You can host content on any ndnSIM node using the `ns3::ndn::FileServer` application. The following code snippet shows how to set up a producer:

`As already mentioned, hosting content does not require any special attention. You can host content on any ndnSIM node, similar as you would do with the ``ns3::ndn::Producer``, by using the ``ns3::ndn::FileServer`` application, as shown in the following sourcecode:
```cplusplus
  // Producer
  ndn::AppHelper producerHelper("ns3::ndn::FileServer");

  // Producer will reply to all requests starting with /prefix
  producerHelper.SetPrefix("/myprefix");
  producerHelper.SetAttribute("ContentDirectory", StringValue("/home/someuser/somedata/"));
  producerHelper.Install(nodes.Get(2)); // install to a node from the nodecontainer
```
This will make the directory  ``/home/someuser/somedata/`` and all contents (including sub-directories) fully available under the ndn prefix ``/myprefix`` within the simulator. 
You can install this producer on as many nodes you want. Just make sure to also provide routing information, the same way as you would for a "plain" ndn-producer.

```cplusplus
  // Optional: Routing  (if and where applicable)
  ndn::GlobalRoutingHelper ndnGlobalRoutingHelper;
  ndnGlobalRoutingHelper.InstallAll();
  ...
  ndnGlobalRoutingHelper.AddOrigins("/myprefix", nodes.Get(2));
  ...
  ndn::GlobalRoutingHelper::CalculateRoutes();
```

## Requesting Content (Consumer)

Similar to the producer setup, the `FileConsumer` is installed on a node. The following code snippet shows how to set up a consumer:
Similar to the example above, the client, also called ``FileConsumer``, needs to be installed on some node:
```cplusplus
  // Consumer
  ndn::AppHelper consumerHelper("ns3::ndn::FileConsumer");
  consumerHelper.SetAttribute("FileToRequest", StringValue("/myprefix/file1.img"));
  
  consumerHelper.Install(nodes.Get(0)); // install to some node from nodelist
```
This simple piece of code will automatically request file1.txt, similar to what ``ns3::ndn::Consumer`` would do.

### Full Example: Basic File Transfer
First, create a data directory in your home directory, and create a file of 10 Megabyte size:
```bash
mkdir ~/somedata
fallocate -l 10M ~/somedata/file1.img
```

Then, create a .cpp file with the following content in your scenario subfolder (see [examples/ndn-file-simple-example1.cpp](https://github.com/ulen2000/Performance-Evaluation-of-Dynamic-Adaptive-Streaming-over-Information-Centric-Networking/examples/ndn-file-simple-example1.cpp)):
```cplusplus
  // setting default parameters for PointToPoint links and channels
  Config::SetDefault("ns3::PointToPointNetDevice::DataRate", StringValue("10Mbps"));
  Config::SetDefault("ns3::PointToPointChannel::Delay", StringValue("10ms"));
  Config::SetDefault("ns3::DropTailQueue::MaxPackets", StringValue("20"));

  // Read optional command-line parameters (e.g., enable visualizer with ./waf --run=<> --visualize
  CommandLine cmd;
  cmd.Parse(argc, argv);

  // Creating nodes
  NodeContainer nodes;
  nodes.Create(3); // 3 nodes, connected: 0 <---> 1 <---> 2

  // Connecting nodes using two links
  PointToPointHelper p2p;
  p2p.Install(nodes.Get(0), nodes.Get(1));
  p2p.Install(nodes.Get(1), nodes.Get(2));

  // Install NDN stack on all nodes
  ndn::StackHelper ndnHelper;
  ndnHelper.SetDefaultRoutes(true);
  ndnHelper.InstallAll();

  // Choosing forwarding strategy
  ndn::StrategyChoiceHelper::InstallAll("/myprefix", "/localhost/nfd/strategy/best-route");

  // Consumer
  ndn::AppHelper consumerHelper("ns3::ndn::FileConsumer");
  consumerHelper.SetAttribute("FileToRequest", StringValue("/myprefix/file1.img"));

  consumerHelper.Install(nodes.Get(0)); // install to some node from nodelist

  // Producer
  ndn::AppHelper producerHelper("ns3::ndn::FileServer");

  // Producer will reply to all requests starting with /prefix
  producerHelper.SetPrefix("/myprefix");
  producerHelper.SetAttribute("ContentDirectory", StringValue("/home/someuser/somedata/"));
  producerHelper.Install(nodes.Get(2)); // install to some node from nodelist

  ndn::GlobalRoutingHelper ndnGlobalRoutingHelper;
  ndnGlobalRoutingHelper.InstallAll();

  ndnGlobalRoutingHelper.AddOrigins("/myprefix", nodes.Get(2));
  ndn::GlobalRoutingHelper::CalculateRoutes();

  Simulator::Stop(Seconds(600.0));

  Simulator::Run();
  Simulator::Destroy();

  NS_LOG_UNCOND("Simulation Finished.");
``` 

You can now compile and start the scenario using
```bash
./waf --run ndn-file-simple-example1
```
Though you will not see any output. We will talk about generating some output based on traces from file transfers in the next section. For now, if you just want to verify that your file transfer is working, use the visualizer:
```bash
./waf --run ndn-file-simple-example1 --vis
```


## Tracing File Transfers

For better understanding of the file transfer process, especially the download speed, file transfer tracers are implemented on the consumer side. These tracers output information about the file download start, receipt of the manifest, and completion of the file download.

 At first, we need to add 3 functions to  our previous scenario. They will deal with printing information to the console:
```cplusplus
// FileDownloadedTrace is called when the file download finished
void
FileDownloadedTrace(Ptr<ns3::ndn::App> app, shared_ptr<const ndn::Name> interestName, double downloadSpeed, long milliSeconds)
{
  std::cout << "Trace: File finished downloading: " << Simulator::Now().GetMilliSeconds () << " "<< *interestName <<
     " Download Speed: " << downloadSpeed/1000.0 << " Kilobit/s in " << milliSeconds << " ms" << std::endl;
}

// FileDownloadedManifestTrace is called when the Manifest has been received, along with the file size in the manifest
void
FileDownloadedManifestTrace(Ptr<ns3::ndn::App> app, shared_ptr<const ndn::Name> interestName, long fileSize)
{
  std::cout << "Trace: Manifest received: " << Simulator::Now().GetMilliSeconds () <<" "<< *interestName << " File Size: " << fileSize << std::endl;
}

// FileDownloadStartedTrace is called when the file download has started (before the manifest has been received)
void
FileDownloadStartedTrace(Ptr<ns3::ndn::App> app, shared_ptr<const ndn::Name> interestName)
{
  std::cout << "Trace: File started downloading: " << Simulator::Now().GetMilliSeconds () <<" "<< *interestName << std::endl;
}
```

Next, we need to make sure to connect them as trace sources:
```cplusplus
  // Connect Tracers
  Config::ConnectWithoutContext("/NodeList/*/ApplicationList/*/FileDownloadFinished",
                               MakeCallback(&FileDownloadedTrace));
  Config::ConnectWithoutContext("/NodeList/*/ApplicationList/*/ManifestReceived",
                               MakeCallback(&FileDownloadedManifestTrace));
  Config::ConnectWithoutContext("/NodeList/*/ApplicationList/*/FileDownloadStarted",
                               MakeCallback(&FileDownloadStartedTrace));
```

You can find the whole sourcecode under [examples/ndn-file-simple-example2-tracers.cpp]https://github.com/ulen2000/Performance-Evaluation-of-Dynamic-Adaptive-Streaming-over-Information-Centric-Networking/examples/ndn-file-simple-example2-tracers.cpp). The output of this looks as follows:

```
Trace: File started downloading: 0 /myprefix/file1.img
Received Manifest! FileSize=10485760, MaxPayload=1449
Resulting Max Seq Nr = 7237
Trace: Manifest received: 41 /myprefix/file1.img File Size: 10485760
Trace: File finished downloading: 468021 /myprefix/file1.img Download Speed: 179.236 Kilobit/s in 468021 ms
```

You might now wonder: Why is the throughput only 179 Kilobit/s, when the link can handle up to 1 Megabit/s? The reason for this is that we only issued an Interest for a fragment once the previous Interest (for a fragment of the manifest) was answered. This results in lot of un-used capacity. The solution to this is pipe-lining Interests, which we also implemented (see next sub-section).


### Pipe-Lining Interests (Enhanced File Consumer)

To fully utilize the link capacity, I recommend using the enhanced version of the `FileConsumer`, the `FileConsumerCbr`. This application issues pipeline Interests based on the average number of data packets per second that the consumer can handle.

 This is achieved by calculating the average number of data packets per second that this consumer can handle (read: ```LinkDownloadSpeedInBytes/PayloadSize```). In addition, we also issue Interests for File/1, File/2, ... before we receive an answere for File/Manifest. While this could lead to unnecessary Interests, in most cases it will speed up the file transfer (as the files transferred are large enough).

Using the Enhanced File Consumer is as simple as replacing ``ns3::ndn::FileConsumer`` with ``ns3::ndn::FileConsumerCbr``, as shown in the following code:

```cplusplus
  // Consumer
  ndn::AppHelper consumerHelper("ns3::ndn::FileConsumerCbr");
  consumerHelper.SetAttribute("FileToRequest", StringValue("/myprefix/file1.img"));
```

You can find the full example here [(https://github.com/ulen2000/Performance-Evaluation-of-Dynamic-Adaptive-Streaming-over-Information-Centric-Networking/examples/ndn-file-simple-example3-enhanced.cpp](https://github.com/ulen2000/Performance-Evaluation-of-Dynamic-Adaptive-Streaming-over-Information-Centric-Networking/blob/master/examples//ndn-file-simple-example3-enhanced.cpp).

The output shows a throughput of 961 Kilobit/s and looks like this:
```
Trace: File started downloading: 0 /myprefix/file1.img
Received Manifest! FileSize=10485760, MaxPayload=1449
Resulting Max Seq Nr = 7237
Trace: Manifest received: 41 /myprefix/file1.img File Size: 10485760
Trace: File finished downloading: 87285 /myprefix/file1.img Download Speed: 961.06 Kilobit/s in 87285 ms
Simulation Finished.
```
This number is still a bit lower than 1 Megabit/s. As the ndn header uses 51 of our available 1500 bytes (Payload size 1449), the resulting goodput is roughly 96.6%  (roughly 966 Kilobit/s), which we almost manage to achieve.

As you can see, ``FileConsumerCbr`` performs much better, and it is the recommended Consumer to use. In addition, ``FileConsumerCbr`` also handles timeouts, re-transmissions and out-of-order packets.


### Testing File Transfers with Real Data
With FileConsumer and FileProducer we are transfering real payloads (data packets) over the simulated network. For some scenarios it might be necessary to also use the file transfered via the simulator (e.g., in a consumer app). We added a method for writeing the received file to the storage with the following command:
```cplusplus
  consumerHelper.SetAttribute("WriteOutfile", StringValue("/home/username/somefile.img"));
```
Feel free to also compare the outfile with the original file by using ```m5sum``` or ```sha1sum```.

While this method helps for debugging/testing, it is not recommended storing those files on permanent storage for long running simulations. Instead, use a ramdisk. In addition, writing files to storage (or ramdisk) does impact performance negatively. 


### Tracing Several File Consumers 
When considering a larger scenario with many clients, it might be required to log the trace output to a file for all clients. This can be easily achieved by using the ``FileConsumerLogTracer`` with the following line of code:
```
  ndn::FileConsumerLogTracer::InstallAll("file-consumer-log-trace.txt");
```

An example of this is provided in [examples/ndn-file-simple-example4-multi.cpp](https://github.com/ulen2000/Performance-Evaluation-of-Dynamic-Adaptive-Streaming-over-Information-Centric-Networking/tree/master/examples/ndn-file-simple-example4-multi.cpp). The output produced by this scenario looks like this:
```
Trace: File started downloading: 0 /myprefix/file1.img
Trace: File started downloading: 0 /myprefix/file1.img
Trace: File started downloading: 0 /myprefix/file1.img
Received Manifest! FileSize=10485760, MaxPayload=1449
Resulting Max Seq Nr = 7237
Trace: Manifest received: 41 /myprefix/file1.img File Size: 10485760
Received Manifest! FileSize=10485760, MaxPayload=1449
Resulting Max Seq Nr = 7237
Trace: Manifest received: 41 /myprefix/file1.img File Size: 10485760
Received Manifest! FileSize=10485760, MaxPayload=1449
Resulting Max Seq Nr = 7237
Trace: Manifest received: 41 /myprefix/file1.img File Size: 10485760
Trace: File finished downloading: 90485 /myprefix/file1.img Download Speed: 927.072 Kilobit/s in 90485 ms
Trace: File finished downloading: 90687 /myprefix/file1.img Download Speed: 925.007 Kilobit/s in 90687 ms
Trace: File finished downloading: 90691 /myprefix/file1.img Download Speed: 924.966 Kilobit/s in 90691 ms
Simulation Finished.
```
As you can see, it becomes harder to track what happens. Needless to say, because of NDNs interest aggregation and caching mechanisms the speed is almost 1 Mbit/s for all three clients.

We can now open file-consumer-log.trace.txt which looks as follows:
```
Time	Node	AppId	InterestName	Event	Info
0	0	0	/myprefix/file1.img	DownloadStarted	NoInfo
0	1	0	/myprefix/file1.img	DownloadStarted	NoInfo
0	2	0	/myprefix/file1.img	DownloadStarted	NoInfo
0.041786	0	0	/myprefix/file1.img	ManifestReceived	FileSize=10485760
0.041786	1	0	/myprefix/file1.img	ManifestReceived	FileSize=10485760
0.041786	2	0	/myprefix/file1.img	ManifestReceived	FileSize=10485760
90.4855	1	0	/myprefix/file1.img	DownloadFinished	Speed=905.343;Time=90.485
90.6877	0	0	/myprefix/file1.img	DownloadFinished	Speed=903.327;Time=90.687
90.6918	2	0	/myprefix/file1.img	DownloadFinished	Speed=903.287;Time=90.691
```

This file provides similaro utput as the console, but it's in a more "processable" format (CSV with \t as separator).



### Hosting Virtual Files
The next step for a larger simulation would be creating multiple producers that host different content with different file sizes. Eventually this will lead to a point where one needs to have multiple directories with multiple files and various sizes on a storage device. While this is a waste of storage (for simulatons), it also slows down simulations (File I/O is THE bottleneck for simulations). 

To counter this problem we implemented a so called ``FakeFileServer``, which takes an additional parameter (``MetaDataFile`` - a CSV file with filenames and filesizes in bytes, separated by a comma) and stores that information (filename + size) in memory, instead of reading actual files from a (slow) storage device.


Here we provide an example of this CSV file, containing 3 files at various sizes:
```
file1.img,1000000
file2.img,1500000
file3.img,2000000
```
Assuming this file is called fake.csv, the producer and consumer configuration will look like this:
```cplusplus
  // Consumer
  ndn::AppHelper consumerHelper("ns3::ndn::FileConsumerCbr");
  
  consumerHelper.SetAttribute("FileToRequest", StringValue("/myprefix/file1.img"));
  consumerHelper.Install(nodes.Get(0));

  consumerHelper.SetAttribute("FileToRequest", StringValue("/myprefix/file2.img"));
  consumerHelper.Install(nodes.Get(1));

  consumerHelper.SetAttribute("FileToRequest", StringValue("/myprefix/file3.img"));
  consumerHelper.Install(nodes.Get(2));

  ...

  // Producer
  ndn::AppHelper producerHelper("ns3::ndn::FakeFileServer");

  // Producer will reply to all requests starting with /prefix
  producerHelper.SetPrefix("/myprefix");
  producerHelper.SetAttribute("MetaDataFile", StringValue("fake.csv"));
  producerHelper.Install(nodes.Get(4)); // install to some node from nodelist
```
The full example is available here: [examples/ndn-file-simple-example5-virtual.cpp](https://github.com/ulen2000/Performance-Evaluation-of-Dynamic-Adaptive-Streaming-over-Information-Centric-Networking/examples/ndn-file-simple-example5-virtual.cpp).


### Summary
In this chapter, we have presented the basics of file transfers in AMuSt-ndnSIM. We can now use this for implementing adaptive multimedia streaming (which is the main purpose of this simulator).

------------------
## 2. Hosting Multimedia Files
For multimedia streaming, we use the concept of Dynamic Adaptive Streaming over HTTP, adapted for the file transfer logic from Part 1. We make use of an extended version of bitmovins [libdash](https://github.com/bitmovin/libdash), which is an open source library for MPEG-DASH. Our version, [AMuSt-libdash](https://github.com/ChristianKreuzberger/AMuSt-libdash/) includes a dummy multimedia player, video playback buffer and several adaptation logics.

Before we talk any more about the client, we describe the process of hosting DASH multimedia content. Hosting DASH content is as simple as telling the ``FileServer`` (see previous chapter) to host a directory with the DASH content. We can basically use any MPEG-DASH compatible video stream, as long as the Media Presentation Description (MPD) file can be parsed by libdash (Note: we only support BASIC MPD structures, with only one period, and no wildcards). 


Alternatively, content can also be hosted by using the ``FakeFileServer`` and a .csv file containing all names and filesizes of all multimedia files. However, the MPD file still needs to be provided by ``FileServer`` (can not be done with ``FakeFileServer``). This process has the advantage of still using a realistic dataset but avoiding the overhead of storing multiple gigabyte of (unnecessary) data on permanent storage.


Yet another method of hosting DASH content is achieved by providing only the information of which representations the video should have. This is handled by ``FakeMultimediaServer``. The big disadvantage of this approach is that all segments of one representation have the exact same size, something that is not necessarily true in real world multimedia applications. However, this method has almost no storage overhead and is most likely the easiest one to start with.

In the following we will show examples for all three approaches, starting with the easiest one (``FakeMultimediaServer``), followed by ``FileServer`` and ``FakeFileServer``.


### Hosting Virtual Content using ``FakeMultimediaServer``
Netflix has [announced](http://techblog.netflix.com/2015/12/per-title-encode-optimization.html) which adaptive streaming representations they have been using in the past in a [blog post](http://techblog.netflix.com/2015/12/per-title-encode-optimization.html). For convenience, here they are:

```
reprId,screenWidth,screenHeight,bitrate
1,320,240,235
2,384,288,375
3,512,384,560
4,512,384,750
10,640,480,1050
11,720,480,1750
20,1280,720,2350
21,1280,720,3000
30,1920,1080,4300
31,1920,1080,5800
``` 

The convenient thing about ``FakeMultimediaServer`` is that we only need to provide those (purposely CSV-formatted) data and some more in a CSV file (netflix_vid1.csv):
```
segmentDuration=2
numberOfSegments=1800
reprId,screenWidth,screenHeight,bitrate
1,320,240,235
2,384,288,375
3,512,384,560
4,512,384,750
10,640,480,1050
11,720,480,1750
20,1280,720,2350
21,1280,720,3000
30,1920,1080,4300
31,1920,1080,5800
```

Config: ``segmentDuration`` specifies the duration of a segment in seconds (2 is a kind-of industry-standard) and ``numberOfSegments`` specifies the number of segments for a video. In this case, the length of the video would be 3600 seconds (60 minutes). The CSV format below contains the representation identifier (reprId, integer), screen width / height (integer), and a bitrate in kbit/s (integer). 

If you want to have several videos with various lengths, you need to provide several CSV files. For convenience, we have uploaded example files for [Netflix here](../representations/). Last but not least, we need to configure a server, which is shown in the code example below:
```cplusplus
  ndn::AppHelper fakeDASHProducerHelper("ns3::ndn::FakeMultimediaServer");

  // This fake multimedia producer will reply to all requests starting with /myprefix/FakeVid1
  fakeDASHProducerHelper.SetPrefix("/myprefix/FakeVid1");
  fakeDASHProducerHelper.SetAttribute("MetaDataFile", StringValue("representations/netflix_vid1.csv"));
  // We just give the MPD file a name that makes it unique
  fakeDASHProducerHelper.SetAttribute("MPDFileName", StringValue("vid1.mpd"));

  fakeDASHProducerHelper.Install(nodes.Get(0));

  // We can install more then one fake multimedia producer on one node:

  // This fake multimedia producer will reply to all requests starting with /myprefix/FakeVid2
  fakeDASHProducerHelper.SetPrefix("/myprefix/FakeVid2");
  fakeDASHProducerHelper.SetAttribute("MetaDataFile", StringValue("representations/netflix_vid2.csv"));
  // We just give the MPD file a name that makes it unique
  fakeDASHProducerHelper.SetAttribute("MPDFileName", StringValue("vid2.mpd"));


  // This fake multimedia producer will reply to all requests starting with /myprefix/FakeVid3
  fakeDASHProducerHelper.SetPrefix("/myprefix/FakeVid3");
  fakeDASHProducerHelper.SetAttribute("MetaDataFile", StringValue("representations/netflix_vid3.csv"));
  // We just give the MPD file a name that makes it unique
  fakeDASHProducerHelper.SetAttribute("MPDFileName", StringValue("vid3.mpd"));

```

You can find the full example, including clients (which we will discuss later), in [AMuSt-ndnSIM/examples/ndn-multimedia-avc-fake-multimedia-server.cpp](https://github.com/ChristianKreuzberger/AMuSt-ndnSIM/blob/master/examples/ndn-multimedia-avc-fake-multimedia-server.cpp).


### Hosting Actual DASH Content with ``FileServer``
While there are plenty of programs out there to create MPEG-DASH streams, reproducability of demos and simulations is a key problem. Therefore we recommend using existing datasets, and we refer to the following two datasets for MPEG-DASH Videos:

* [DASH/AVC Dataset](http://www-itec.uni-klu.ac.at/dash/?page_id=207) by Lederer et al. [[PDF]](http://www-itec.uni-klu.ac.at/bib/files/p89-lederer.pdf) [[Bib]](http://www-itec.uni-klu.ac.at/bib/index.php?key=Mueller2012&bib=itec.bib) [[Website]](http://www-itec.uni-klu.ac.at/dash/?page_id=207) (we recommend using the older 2012 version for testing purpose)
* [DASH/SVC Dataset](http://concert.itec.aau.at/SVCDataset/) by Kreuzberger et al. [[PDF]](http://www-itec.uni-klu.ac.at/bib/files/dash_svc_dataset_v1.05.pdf) [[Bib]](http://www-itec.uni-klu.ac.at/bib/index.php?key=Kreuzberger2015a&bib=itec.bib) [[Website]](http://concert.itec.aau.at/SVCDataset/)

In the following, we will provide an example for multimedia streaming with DASH/AVC and DASH/SVC, based on those two datasets. We selected the Big Buck Bunny movie to achieve compareable results.

#### DASH/AVC
**Note:** This will require roughly 4 Gigabyte of diskspace
We are going to download the BigBuckBunny movie from the DASH/AVC Dataset, segment length 2 seconds.
First of all, download the video files from [here](http://www-itec.uni-klu.ac.at/ftp/datasets/mmsys12/BigBuckBunny/bunny_2s/):
```bash
cd ~
mkdir -p multimediaData/AVC/BBB/
cd multimediaData/AVC/BBB/

wget -r --no-parent --cut-dirs=5 --no-host --reject "index.html" http://www-itec.uni-klu.ac.at/ftp/datasets/mmsys12/BigBuckBunny/bunny_2s/
```
this folder should also contain the MPD file ``BigBuckBunny_2s_isoffmain_DIS_23009_1_v_2_1c2_2011_08_30.mpd``, if not, you can download the
 [MPD file](http://www-itec.uni-klu.ac.at/ftp/datasets/mmsys12/BigBuckBunny/bunny_2s/BigBuckBunny_2s_isoffmain_DIS_23009_1_v_2_1c2_2011_08_30.mpd) manually.
```bash
wget http://www-itec.uni-klu.ac.at/ftp/datasets/mmsys12/BigBuckBunny/bunny_2s/BigBuckBunny_2s_isoffmain_DIS_23009_1_v_2_1c2_2011_08_30.mpd
```
Open the MPD in an editor of your choice, and locate the ``<BaseURL>`` tag. Change
```xml
<BaseURL>http://www-itec.uni-klu.ac.at/ftp/datasets/mmsys12/BigBuckBunny/bunny_2s/</BaseURL>
```
to
```xml
<BaseURL>/myprefix/AVC/BBB/</BaseURL>
```
Furthermore, we recommend renaming the file to a name of your choice, e.g., BBB-2s.mpd.
```bash
mv BigBuckBunny_2s_isoffmain_DIS_23009_1_v_2_1c2_2011_08_30.mpd BBB-2s.mpd
```

Last but not least, we would like to separate the mpd file from the segments. Therefore we move the mpd file up by one folder (to ~/multimediaData/AVC/):

```bash
mv BBB-2s.mpd ../
```

Finally, your directories should look like this:

 * multimediaData/AVC/
    * BBB-2s.mpd
    * BBB/
        * bunny_2s_50kbit/*.m4s
        * bunny_2s_100kbit/*.m4s
        * ...
        * bunny_2s_8000kbit/*.m4s

#### DASH/SVC
**Note:** This will require roughly 1 Gigabyte of diskspace
We are going to download the BigBuckBunny movie from the DASH/SVC Dataset, with a segment length of 2 seconds, and no temporal scalability. First of all, download the video files from [here](http://concert.itec.aau.at/SVCDataset/dataset/BBB/III/segs/).
```bash
cd ~
mkdir -p multimediaData/SVC/BBB/
cd multimediaData/SVC/BBB/
mkdir III
cd III
wget -r --no-parent --no-host-directories --no-directories http://concert.itec.aau.at/SVCDataset/dataset/BBB/III/segs/1080p/
```
Second, download the [MPD file](http://concert.itec.aau.at/SVCDataset/dataset/mpd/BBB-III.mpd).
```bash
cd .. # go one directory up
wget http://concert.itec.aau.at/SVCDataset/dataset/mpd/BBB-III.mpd
```
Open the MPD in an editor of your choice, and locate the ``<BaseURL>`` tag. Change
```xml
<BaseURL>http://concert.itec.aau.at/SVCDataset/dataset/BBB/III/segs/1080p/</BaseURL>
```
to
```xml
<BaseURL>/myprefix/SVC/BBB/III/</BaseURL>
```



#### Creating a Server with Above Datasets
Finally, creating a server is as simple as:
```cplusplus
  // Producer
  ndn::AppHelper producerHelper("ns3::ndn::FileServer");

  // Producer will reply to all requests starting with /myprefix
  producerHelper.SetPrefix("/myprefix");
  producerHelper.SetAttribute("ContentDirectory", StringValue("/home/username/multimediaData"));
  producerHelper.Install(nodes.Get(0)); // install to some node from nodelist

```


See [examples/ndn-multimedia-avc-server.cpp](https://github.com/ulen2000/Performance-Evaluation-o-Dynamic-Adaptive-Streaming-over-Information-Centric-Networking/examples/ndn-multimedia-avc-server.cpp) for the full example for AVC. 

### Hosting Real Content using FakeFileServer
This part assumes that you followed the DASH Dataset part just before this section. You probably noticed that a good portion of storage space is gone. To counter this problem, we provide ``FakefileServer`` with a CSV file for BBB (AVC), which you can get [here](../datasets/segmentlist/). This file contains a list of all segments and their size. We generated this list by first downloading the dataset and then using some Linux command line magic (mainly sed and awk):
```shell
find . -type f -exec du -a {} + | sed 's/[ \t]/,/g' | sed 's/\.\///g' | awk -F $',' ' { t = $1; $1 = $2; $2 = t; print; } ' OFS=$',' > list_of_files.csv
```

Now we need to configure two producers as follows

 * ``FileServer`` hosts the actual MPD file of BBB at /myprefix/AVC
 * ``FakeFileServer``hosts the virtual segments of BBB at /myprefix/AVC/BBB


```cplusplus
  // Producer responsible for hosting the MPD file
  ndn::AppHelper mpdProducerHelper("ns3::ndn::FileServer");

  // Producer will reply to all requests starting with /myprefix/AVC/ and hosts the mpd file there
  mpdProducerHelper.SetPrefix("/myprefix/AVC");
  mpdProducerHelper.SetAttribute("ContentDirectory", StringValue("/home/someuser/multimediaData"));
  mpdProducerHelper.Install(nodes.Get(0)); // install to some node from nodelist

  // Producer responsible for hosting the virtual segments
  ndn::AppHelper fakeSegmentProducerHelper("ns3::ndn::FakeFileServer");

  // Producer will reply to all requests starting with /myprefix/AVC/BBB/ and hosts the virtual segment files there
  fakeSegmentProducerHelper.SetPrefix("/myprefix/AVC/BBB");
  fakeSegmentProducerHelper.SetAttribute("MetaDataFile", StringValue("dash_dataset_avc_bbb.csv"));
  fakeSegmentProducerHelper.Install(nodes.Get(0)); // install to some node from nodelist
```

See [examples/ndn-multimedia-avc-fake-server.cpp](https://github.com/ulen2000/Performance-Evaluation-of-Dynamic-Adaptive-Streaming-over-Information-Centric-Networking/examples/ndn-multimedia-avc-fake-server.cpp) for the full example.



## 3. Adaptive Multimedia Streaming (DASH)

### Basics
For the next part of the tutorial, we will move on to adaptive multimedia streaming, using MPEG-DASH. We have implemented a simple DASH client and a DASH server. The DASH client is built upon the enhanced File Consumer, and it additionally maintains a buffer, and it can decide which quality/bitrate to request next. The DASH server is built upon the File Server, and it automatically generates the MPEG-DASH MPD (Media Presentation Description) for the consumer.

In order to use the DASH client and server, we first need to prepare some multimedia data. We will use the Big Buck Bunny dataset, which can be downloaded [here](http://www-itec.uni-klu.ac.at/ftp/datasets/DASHDataset2014/BigBuckBunny/). 

We need to convert this dataset from the original MP4 format to the MPEG-DASH format. The easiest way is to use the [MP4Box](https://gpac.wp.mines-telecom.fr/mp4box/dash/) tool. The following bash script can be used to convert all .mp4 files into .m4s files (with a segment duration of 2 seconds):
```bash
#!/bin/bash
mkdir -p dash/
for f in `ls *.mp4`; do
   MP4Box -dash 2000 -frag 2000 -rap -segment-name 'dash/'$f'_' $f
done
```
This should result in a lot of .m4s files within the dash/ subdirectory, and an MPD file (dash.mpd) in your current directory.

Next, we need to create a .csv file which describes the bitrates of the videos. The first column of the .csv file contains the name of the representation (in the MPD file), and the second column contains the bitrate of the representation in Kilobit per second.

Here is an example of a .csv file:
```
bunny_183kbit/bunny_183kbit,183
bunny_553kbit/bunny_553kbit,553
bunny_1248kbit/bunny_1248kbit,1248
bunny_2224kbit/bunny_2224kbit,2224
bunny_4000kbit/bunny_4000kbit,4000
bunny_8000kbit/bunny_8000kbit,8000
```
Save this file as dash/dash_videorate.csv. Your multimedia dataset directory should now look like this:
```
dash/
  bunny_183kbit_dashinit.mp4
  bunny_183kbit_dash1.m4s
  ...
  bunny_8000kbit_dashinit.mp4
  bunny_8000kbit_dash1.m4s
  ...
  dash_videorate.csv
dash.mpd
```
### Hosting Content (DASH Server)
Hosting content is as easy as hosting files. The only difference is that you have to use ``ns3::ndn::DashServer`` instead of ``ns3::ndn::FileServer``, and that you need to provide the name of the .mpd file and the .csv file (relative to your content directory):

```cplusplus
  // Producer
  ndn::AppHelper producerHelper("ns3::ndn::DashServer");
  producerHelper.SetPrefix("/myprefix");
  producerHelper.SetAttribute("ContentDirectory", StringValue("/home/someuser/somedata/"));
  producerHelper.SetAttribute("MPDFileToServe", StringValue("/dash/dash.mpd"));
  producerHelper.SetAttribute("MP

### Multimedia Streaming (DASH over ICN)

Adaptive multimedia streaming is a complex topic. In this tutorial, we will only cover how to use it in here. We will not cover the actual implementation and details of the DASH client and server. For more information on this topic, please refer to the [original publication](https://conferences.sigcomm.org/sigcomm/2016/pdf/papers/p673.pdf).

Before we start, we need to point out that our multimedia dataset is stored in the following directory structure:
```bash
~/multimediaData
~/multimediaData/video1/video1.mpd
~/multimediaData/video1/1.m4s
~/multimediaData/video1/2.m4s
...
~/multimediaData/video2/video2.mpd
...
```
The .mpd file is the DASH manifest file, and the .m4s files are segments of the video. Each .m4s file is a representation (with a specific bitrate) of a specific video segment (of a specific time period). We assume that the manifest file (mpd) and the video segments are already generated and available in this structure. 

### Hosting Multimedia Content (DASH-Server)

Hosting the multimedia content is very similar to hosting files. We will use the `ns3::ndn::FileServer` application to host the multimedia content, as shown in the following code:

```cplusplus
  // Producer
  ndn::AppHelper producerHelper("ns3::ndn::FileServer");

  // Producer will reply to all requests starting with /prefix
  producerHelper.SetPrefix("/myprefix");
  producerHelper.SetAttribute("ContentDirectory", StringValue("/home/someuser/multimediaData/"));
  producerHelper.Install(nodes.Get(2)); // install to a node from the nodecontainer
```

This will make the directory `/home/someuser/multimediaData/` and all contents (including sub-directories) fully available under the ndn prefix `/myprefix` within the simulator.

### Requesting Multimedia Content (DASH-Client)

In this section, we will use the `ns3::ndn::FileConsumerCbr::MultimediaConsumer` application to request the multimedia content. This application is specifically designed to deal with DASH streaming. The DASH client will automatically request the segments based on the DASH algorithm. Here is a simple example of how to use the DASH client:

```cplusplus
  // Consumer
  ndn::AppHelper consumerHelper("ns3::ndn::FileConsumerCbr::MultimediaConsumer");
  consumerHelper.SetAttribute("MpdFileToRequest", StringValue("/myprefix/video1/video1.mpd"));
  
  consumerHelper.Install(nodes.Get(0)); // install to some node from nodelist
```

In this example, the DASH client will request the mpd file of `video1` and start streaming based on the DASH algorithm. 

### Full Example: Basic Multimedia Streaming

First, we need to prepare our multimedia dataset. We can use any video and convert it into DASH format using MP4Box. Here is an example of how to do it:

```bash
# First, create a multimedia directory in your home directory
mkdir ~/multimediaData

# Second, convert your video into DASH format
MP4Box -dash 2000 -rap -segment-name '%s' -url-template -out ~/multimediaData/video1/video1.mpd yourvideo.mp4
```

In this example, `yourvideo.mp4` is the video you want to stream. The `2000` parameter is the segment duration in milliseconds.

A full example can be found in the [examples/ndn-dash-simple-example.cpp](https://github.com/ulen2000/Performance-Evaluation-of-Dynamic-Adaptive-Streaming-over-Information-Centric-Networking/blob/master/examples/ndn-dash-simple-example.cpp) file.

## 4. Generating Large Networks using BRITE

In order to generate larger networks and test the scalability of your application, you can use the BRITE topology generator. This allows you to generate complex network topologies based on different models, such as Barabási–Albert or Waxman models.

We will not cover BRITE in detail, but we encourage you to check out the [BRITE User Manual](https://www.cs.bu.edu/brite/user_manual/node14.html).
In this section, we will show you how to generate larger networks using the BRITE topology generator. BRITE can be used to generate different kinds of topologies, including hierarchical, router-level, and AS-level topologies. 

You can use the `BriteTopologyHelper` to generate the network topology and install the NDN stack:

```cplusplus
BriteTopologyHelper briteth;
briteth.BuildBriteTopology(nd
```

Full example here: [examples/ndn-multimedia-brite-example1.cpp](https://github.com/ulen2000/Performance-Evaluation-of-Dynamic-Adaptive-Streaming-over-Information-Centric-Networking/examples/ndn-multimedia-brite-example1.cpp).

------------------
## Docker
I have packaged the environment and uploaded it to [Docker Hub](docker pull 27718842/dash\_ndnsim\_samus).

All experiment running code is consolidated into this file: ccr_scenario.py. 
One could adapt the settings from lines 490–530 (e.g., costs, delay, capacity used in this paper). 
After downloading the dataset, we should adapt line 303 in ./itec-ndn/scenarios/ccr_scenario.cc to point to the dataset directory.  


------------------






## Citation

Christian Kreuzberger, Daniel Posch, Hermann Hellwagner "AMuSt Framework - Adaptive Multimedia Streaming Simulation Framework for ns-3 and ndnSIM", https://github.com/ChristianKreuzberger/AMust-Simulator/

Bibtex:
```
@misc{kreuzberger2016amust,
  title={AMuSt Framework - Adaptive Multimedia Streaming Simulation Framework for ns-3 and ndnSIM},
  author={Kreuzberger, Christian and Posch, Daniel and Hellwagner, Hermann},
  journal={https://github.com/ChristianKreuzberger/amust-simulator},
  year={2016}
}
```



