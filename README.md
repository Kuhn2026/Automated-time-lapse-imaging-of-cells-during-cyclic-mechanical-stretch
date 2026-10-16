# Automated-time-lapse-imaging-of-cells-during-cyclic-mechanical-stretch

This repository contains the source code used for the automated acquisition and processing of time-lapse microscopy data of cells subjected to cyclic mechanical stretch.

Overview:

The repository includes:

- Python code for the automated control of time-lapse video acquisition.
- MATLAB code for post-processing the recorded image sequences to generate focused images from the acquired data.
- Sample data-set

The following DOI (https://doi.org/10.5281/zenodo.23413589) provides access to a sample dataset showing wound closure in lung epithelial cells. 
The raw videos can be used to test the focus selection algorithm in MATLAB. The following DOI (https://doi.org/10.5281/zenodo.23417450) provides access to a sample dataset with fewer, scattered single cells (to test the code under different conditions). In addition, the videos illustrate the conditions required for successful focus selection, including sufficient illumination, a uniform and dense cell distribution, and adequate image resolution.

# Requirements for Use

The entire setup is designed to operate on a Windows system.

All components of the FX stretching system must be present, including the control software. The system should be set up and operated in accordance with the user manual.

To operate and control the cameras, the official **IDS peak** software from IDS Imaging Development Systems and a USB 3.0 connection are required.

A version compatible with both the cameras in use and the computer’s operating system must be installed in advance. The Python script used for automated time-lapse documentation also requires a suitable software environment, including Python and the necessary Python libraries.

Videos are recorded in MP4 format. For longer recording sessions, sufficient storage space must be available. The specifications of the computer used can serve as a reference:

- Installed RAM: 16 GB
- Mass storage: 500 GB SSD
- CPU: Intel Core i7-4770S (4 cores, 8 threads)
- Operating system: Windows 11

MATLAB is required for automated in-focus image selection. Please ensure that MATLAB is installed before use.  

# Installation Checklist

- Windows 11 (64-bit) installed and configured
- Compatible Python version installed (e.g., Python 3.12.5)
- IDS peak, including the Python bindings, installed
- Cameras detected in IDS peak Cockpit
- Short test recordings completed successfully
- Video quality and available storage space checked

# Operating Instructions for Automated Time-Lapse Documentation

1. Ensure that the BioFlex 6-well plate is properly positioned and sealed in the stretching device.
2. Start an audio sequence on the computer to activate the LEDs.
3. Position the cameras above the wells.
4. Adjust the focus using the Z-axis.
5. Start the stretching process in the FlexCell software.
6. Pause the stretching process and readjust the focus. Ensure that the focus is set on the membrane in its relaxed position.
7. Resume the stretching process.
8. Close the camera software and stop the audio file.
9. Start the Python script.
10. The user will be prompted to specify the following parameters:
   - Number of active cameras
   - Recording duration per video, in seconds
   - Interval between recordings, in seconds
   - Total recording time, in minutes
11. Once these parameters have been entered, the time-lapse documentation will run automatically.
All recorded videos are saved in a folder named **"Videos"**.

# Operating Instructions for Automated in-foxcus Image Selection

# Instructions for Selecting In-Focus Frames Using MATLAB

1. Ensure that all videos from a single recording session are saved in a separate folder.
2. Download the MATLAB code from the GitHub repository and save it in your MATLAB working directory.
3. Run the MATLAB code.
4. When prompted, specify the source folder containing the video files.
5. Monitor the processing progress in the MATLAB Command Window.
6. Once the code has finished running, a new folder named **“FocusedFrames”** will be created inside the specified source folder. This folder contains the selected in-focus images extracted from the videos.
7. In addition, a text file and an Excel file will be generated. Both files contain the ID of the selected frame and its corresponding sharpness score for each video.

The selected images are now ready for subsequent analyses.

