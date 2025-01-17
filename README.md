# Decode Passport Scanner

![the box image](the_box/the_box_elements.jpg)

In this repository you will find everything you need to create your own Passport Scanner. It could be used stand alone to read data of the NFC chip inside a passport, but it's full potential is to read the data and then make it available from transfer to the [Decode Amsterdam PWA](https://github.com/Amsterdam/decode_amsterdam_pwa).

**Important note:**
This project was created as a proof of concept prototype. There for it was never meant to be used on a large scale but for demonstration purposes only. There are some outdated libraries needed to get ot running, e.g. pycrypto which showes a security message here on GitHub.

In this repository you will find the following:

* Code for communication with different hardware elements
* Code for communication with the [Decode Session Manager](https://github.com/Amsterdam/decode_session_manager)
* Documentation for building a physical box to combine all hardware elements 

The code and hardware has only been tested using Ubuntu 18.04.


## Hardware

### NFC Reader

Before we start it is good to mention there is a specific library being used to handle all the complicated interaction between a NFC scanner and the NFC chip in a passport. This library, PyPassport, needed to be modified slightly in order to work with the NFC readers we had available. This modified version can be found as a submodule of this project and [here](https://github.com/sguldemond/pypassport). The library it self is using the standard of Machine Readable Travel Documents (MRTD) as defined in [ICAO Doc 9303](https://www.icao.int/publications/pages/publication.aspx?docnum=9303). 

The NFC reader we ended up using is the [ACS ACR1252U-M1](https://www.acs.com.hk/en/products/342/acr1252u-usb-nfc-reader-iii-nfc-forum-certified-reader/), supported by the [CCID driver](https://ccid.apdu.fr/).

#### Testing the NFC Reader

List all USB devices: `$ lsusb`

Using the [PCSC-lite](https://pcsclite.apdu.fr/) daemon `pcscd` you can check if the driver is compatible with a NFC scanner:
`$ sudo pcscd -f -d`

Start PCSC in the background:
`$ service pcscd start`

Using [pcsc-tools](http://ludovic.rousseau.free.fr/softwares/pcsc-tools/) chips on the scanner can be read, PCSC needs te be started for this:
`$ pcsc_scan`


### Camera

The camera is used to read the MRZ (Machine Readable Zone) on a passport which on its turn it used to decrypt the data on the passport's NFC chip.

Any modern webcam will do, as long as it has a decent resolution to perform OCR. We ended up using the [Razer Kiyo](https://www.razer.com/gaming-broadcaster/razer-kiyo). We modified it by removing the stand so only the camera was left, this way it fit nicely in the box.


### The Box

The box combines all the hardware in to one physical element. For our setup we used three more elements to complete the setup:
* PC, Intel NUC i5 with BX500 120GB SSD & Crucial 4GB DDR RAM
* Monitor, [Seeed 10.1 inch LCD display](https://www.seeedstudio.com/10-1-Inch-LCD-Display-1366x768-HDMI-VGA-NTSC-PAL-p-1586.html)
* Keyboard, [Logitech K400 Wireless Touch](https://www.logitech.com/en-us/product/wireless-touch-keyboard-k400r)

In the folder `the_box` you can a Sketch Up file of this render:

![the box image](the_box/the_box_render.png)

All the AutoCAD files (`.dxf`) to laser cut the different pieces are also available there.

## Requirements

- Tesseract for PassportEye OCR:
```
# apt install tesseract-ocr
# apt install libtesseract-dev
```
(install guide: https://github.com/tesseract-ocr/tesseract/wiki)

## Setup

Install virtualenv:
```
$ pip install virtualenv
```

Setup and activate Python 2.7 virtual environment:
```
$ virtualenv --python=/usr/bin/python venv
$ source venv/bin/activate
```

Install the requirements:
```
(venv) $ pip install -r requirements.txt
```

Now you can install the modified version of PyPassport (see Appendix for details about the modifications):
```
$ git clone https://github.com/sguldemond/pypassport
$ cd pypassport
(venv) $ pip install .
```

Create config file:
```
$ nano config.py
```
Input for file:
```
SERVER_CONFIG = {
  "dev": "http://localhost:5000",
  "prod": "https://api.decode.amsterdam"
}
```

## Run

```
$ (venv) python frontend.py
```

This should start connecting the de Session Manager and eventually show the first screen.


## Appendix

### PyPassport modifications

- [Origin](https://code.google.com/archive/p/pypassport/)
- [GitHub mirror](https://github.com/andrew867/epassportviewer)

The reading of the NFC chip is supported by the PyPassport python library. This project has been actively investigated. It takes some time to get all the needed software installed since main project was last updated 4 years ago. During the development we got stuck at reading the passport information. We eventually made some changes to the source code to get it working.

Changing protocol from T0 to T1, based on experiments done using [gscriptor](ludovic.rousseau.free.fr/softwares/pcsc-tools/)
pypassport > reader.py > class PcscReader > def connect: 
```
# self._pcsc_connection.connect(self.sc.scard.SCARD_PCI_T0)
self._pcsc_connection.connect(self.sc.scard.SCARD_PCI_T1)
```

Adding this line to
pypassport > doc9303 > mrz.py > class MRZ > def _checkDigitsTD1 & def _checkDigitsTD2:
```
mrz = self._mrz
```

When running the 'EPassport.readPassport' method we came across an error with this message:
```
('Data not found:', '6F63')
```
This is most likely a merging of the '6F' and '63' which are the locations of DG15 (Public Keys) and DG3 (Finger Print) respectively on the LDS (Logical Data Structure). For some reason these two tags are stored conjoined in the common file, which contains a list of available DG's. This list can be read using 'EPassport.readCom'.

I added this code to 'readCom', which does not yet fix the whole problem:
```
# Temp fix for 6F63 Data not found issue
double_tag = False
ef_com = self["Common"]["5C"]
for tag in ef_com:
  if tag == "6F63":
  ef_com.append("6F")
  ef_com.append("63")
  double_tag = True
        
  if double_tag:
  ef_com.remove("6F63")
        
 # print(ef_com)
 ###
```

For some reason DG15 returns empty and for DG3 the 'securty status is not satisfied'.
For now this can be skipped, with the info from DG15 (public keys) the info on the NFC can be varified as valid, but this is not relevant for us at this moment. We also don't need DG3 (Finger Print).

### ICAO Doc 9303

The standard around Machine Readable Travel Documents can be found at [here](https://www.icao.int/publications/pages/publication.aspx?docnum=9303)
