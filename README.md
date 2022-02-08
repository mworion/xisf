<a id="xisf"></a>

# xisf

(Incomplete) XISF Encoder/Decoder (see https://pixinsight.com/xisf/).

This implementation is not endorsed nor related with PixInsight development team.

Copyright (C) 2021 Sergio Díaz, sergiodiaz.eu

This program is free software: you can redistribute it and/or modify it
under the terms of the GNU General Public License as published by the
Free Software Foundation, version 3 of the License.

This program is distributed in the hope that it will be useful, but WITHOUT
ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or
FITNESS FOR A PARTICULAR PURPOSE.  See the GNU General Public License for
more details.

You should have received a copy of the GNU General Public License along with
this program.  If not, see <http://www.gnu.org/licenses/>.

<a id="xisf.XISF"></a>

## XISF Objects

```python
class XISF()
```

Implements an *incomplete* (attached images only) baseline XISF Decoder and a simple baseline Encoder.
It parses metadata from Image and Metadata XISF core elements. Image data is returned as a numpy ndarray 
(using the "channels-last" convention by default). 

What's supported: 
- Monolithic XISF files only
    - XISF blocks with attachment block locations (neither inline nor embedded block locations as required 
      for a complete baseline decoder)
    - Planar pixel storage models, *however it assumes 2D images only* (with multiple channels)
    - UInt8/16/32 and Float32/64 pixel sample formats
    - Grayscale and RGB color spaces     
- Decoding:
    - multiple Image core elements from a monolithic XISF file
    - Support all standard compression codecs defined in this specification for decompression (zlib/lz4[hc]+
      byte shuffling)
- Encoding:
    - Single image core element
    - Uncompressed data blocks only       
- "Atomic" properties only (Scalar, Strings, TimePoint)
- Metadata and FITSKeyword core elements

What's not supported (at least by now):
- Read pixel data from XISF blocks with inline or embedded block locations
- Read pixel data in the normal pixel storage models
- Read pixel data in the planar pixel storage models other than 2D images
- Complex, Vector, Matrix and Table properties
- Any other not explicitly supported core elements (Resolution, Thumbnail, ICCProfile, etc.)

Usage example:
```
>>> from xisf import XISF
>>> import matplotlib.pyplot as plt
>>> xisf = XISF("file.xisf")
>>> file_meta = xisf.get_file_metadata()    
>>> file_meta
>>> ims_meta = xisf.get_images_metadata()
>>> ims_meta
>>> im_data = xisf.read_image(0)
>>> plt.imshow(im_data)
>>> plt.show()
>>> XISF.write("output.xisf", im_data, ims_meta[0], file_meta)
```

If the file is not huge and it contains only an image (or you're interested just in one of the 
images inside the file), there is a convenience method for reading the data and the metadata:
```
>>> from xisf import XISF
>>> import matplotlib.pyplot as plt    
>>> im_data = XISF.read("file.xisf")
>>> plt.imshow(im_data)
>>> plt.show()
```

The XISF format specification is available at https://pixinsight.com/doc/docs/XISF-1.0-spec/XISF-1.0-spec.html

<a id="xisf.XISF.__init__"></a>

#### \_\_init\_\_

```python
def __init__(fname)
```

Opens a XISF file and extract its metadata. To get the metadata and the images, see get_file_metadata(),
get_images_metadata() and read_image().

**Arguments**:

- `fname` - filename
  

**Returns**:

  XISF object.

<a id="xisf.XISF.get_images_metadata"></a>

#### get\_images\_metadata

```python
def get_images_metadata()
```

Provides the metadata of all image blocks contained in the XISF File, extracted from
the header (<Image> core elements). To get the actual image data, see read_image().

It outputs a dictionary m_i for each image, with the following structure:

```
m_i = { 
    'geometry': (width, height, channels), # only 2D images (with multiple channels) are supported
    'location': (pos, size), # used internally in read_image()
    'dtype': np.dtype('...'), # derived from sampleFormat argument
    'compression': (codec, uncompressed_size, item_size), # optional
    'key': 'value', # other <Image> attributes are simply copied 
    ..., 
    'FITSKeywords': { <fits_keyword>: fits_keyword_values_list, ... }, 
    'XISFProperties': { <xisf_property_name>: property_dict, ... }
}

where:

fits_keyword_values_list = [ {'value': <value>, 'comment': <comment> }, ...]
property_dict = {'id': <xisf_property_name>, 'type': <xisf_type>, 'value': property_value, ...}
```

**Returns**:

  list [ m_0, m_1, ..., m_{n-1} ] where m_i is a dict as described above.

<a id="xisf.XISF.get_file_metadata"></a>

#### get\_file\_metadata

```python
def get_file_metadata()
```

Provides the metadata from the header of the XISF File (<Metadata> core elements).

**Returns**:

  dictionary with one entry per property: { <xisf_property_name>: property_dict, ... }
  where:
  ```
  property_dict = {'id': <xisf_property_name>, 'type': <xisf_type>, 'value': property_value, ...}
  ```

<a id="xisf.XISF.get_metadata_xml"></a>

#### get\_metadata\_xml

```python
def get_metadata_xml()
```

Returns the complete XML header as a xml.etree.ElementTree.Element object.

**Returns**:

- `xml.etree.ElementTree.Element` - complete XML XISF header

<a id="xisf.XISF.read_image"></a>

#### read\_image

```python
def read_image(n=0, data_format='channels_last')
```

Extracts an image from a XISF object.

**Arguments**:

- `n` - index of the image to extract in the list returned by get_images_metadata()
- `data_format` - channels axis can be 'channels_first' or 'channels_last' (as used in
  keras/tensorflow, pyplot's imshow, etc.), 0 by default.
  

**Returns**:

  Numpy ndarray with the image data, in the requested format (channels_first or channels_last).

<a id="xisf.XISF.read"></a>

#### read

```python
@staticmethod
def read(fname, n=0, image_metadata={}, xisf_metadata={})
```

Convenience method for reading a file containing a single image.

**Arguments**:

- `fname` _string_ - filename
- `n` _int, optional_ - index of the image to extract (in the list returned by get_images_metadata()). Defaults to 0.
- `image_metadata` _dict, optional_ - dictionary that will be updated with the metadata of the image.
- `xisf_metadata` _dict, optional_ - dictionary that will be updated with the metadata of the file.
  

**Returns**:

- `[np.ndarray]` - Numpy ndarray with the image data, in the requested format (channels_first or channels_last).

<a id="xisf.XISF.write"></a>

#### write

```python
@staticmethod
def write(fname, im_data, image_metadata={}, xisf_metadata={})
```

Writes an image (numpy array) to a XISF file.

**Arguments**:

- `fname` - filename (will overwrite if existing)
- `im_data` - numpy ndarray with the image data
- `image_metadata` - dict with the same structure described for m_i in get_images_metadata().
  Only 'FITSKeywords' and 'XISFProperties' keys are actually written, the rest are derived from im_data.
- `xisf_metadata` - dict with the same structure returned by get_file_metadata()
  

**Returns**:

  Nothing

