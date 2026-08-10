This dataset consists of the content and sensory data of 360-degree videos to HMD.
The following is the file structure:
360dataset
   |---content
   |     |---saliency
   |	 |      |---videoname_saliency.mp4
   |     |
   |	 |---motion
   |     |      |---videoname_motion.mp4
   |
   |---sensory
	 |---  raw
	 |      |---videoname_user00_raw.csv
	 |
	 |---orientation
	 |      |---videoname_user00_orientation.csv
	 |
	 |---  tile
	        |---videoname_user00_tile.csv

The folder information:
- saliency: contains the videos with the saliency map of each frame based on Cornia's work[1], where the saliency maps indicate the attraction level of the original video frame.

- motion: contains the videos with the optical flow analyzed from each consecutive frames, where the optical flow indicates the relative motions between the objects in 360-degree videos and the viewer.

- raw: contains the raw sensing data (raw x, raw y, raw z, raw yaw, raw roll, and raw pitch) with timestamps captured when the viewers are watching 360-degree videos using OpenTrack[2].

- orientation: contains the orientation data (raw x, raw y, raw z, raw yaw, raw roll, and raw pitch), which have been aligned with the time of each frame of the video. The calibrated orientation data (cal. yaw, cal. pitch, and cal. roll) are provided as well.

- tile: contains the tile numbers overlapped with the Field-of-View (FoV) of the viewer according to the orientation data, where the tile size is 192x192. Knowing the tiles that are overlapped with the FoV of the viewer is useful for optimizing 360-degree video to HMD streaming system, for example, the system can only stream those tiles to reduce the required bandwidth or allocate higher bitrate to those tiles for better user experience.

[1] M. Cornia, L. Baraldi, G. Serra, and R. Cucchiara. 2016. A Deep Multi-Level Network for Saliency Prediction. In Proc. of International Conference on Pattern Recognition (ICPR'16). Cancun, Mexico.
[2] OpenTrack: head tracking software. (2017). https://github.com/opentrack/opentrack.

