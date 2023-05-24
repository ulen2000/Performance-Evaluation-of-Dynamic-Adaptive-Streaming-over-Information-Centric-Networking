# Performance-Evaluation-of-Dynamic-Adaptive-Streaming-over-Information-Centric-Networking

*Have packaged the environment and uploaded it to \textbf{Docker Hub}\footnote{docker pull 27718842/dash_ndnsim_samus }.*

Within this tutorial, we will try to cover  examples like file-transfers, up to building a large example with several multimedia streaming clients(DASH). We assume that multimedia data(dataset) are stored in /home/someuser/multimediaData, whereas you might want to replace someuser with your username.

This tutorial is organized as follows: Section 1 explains how the file transfers are organized and implemented. Section 2 explains how adaptive multimedia streaming can be used. Last but not least, we will show how to generate a large network using BRITE in Section 3.

Throughout the tutorial we place any code in the ndnSIM/ns-3/scratch folder, and that executing the scenarios by using ./waf --run scenario_name within the ndnSIM/ns-3 folder.



