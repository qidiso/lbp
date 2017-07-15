# LBP Library

A collection of Local Binary Pattern (LBP) algorithms.

## OLBP

The implementation requires single-channel images for input and allows for any
radius and number of neighborhood pixels. The neighborhood is computed using the
formula in the cited paper, with the elements visited
clockwise. I have supplied a set of specializations for a more efficient
traversal of neighbors: *(8,1)*, *(10,2)*, *(12,2)*, *(16,2)*:

    // One plane frame:
    lbp::olbp_t< unsigned char, 1, 8 > op;
    auto result = op (frame);

Last but not least, the central pixel for a neighborhood and the neighborhood
itself can belong to different planes, e.g.:

    // E.g., split a 3-plane, RGB frame:
    vector< Mat > planes;
    split (frame, planes);
    
    // Apply pperator on plane 0 neighbors, against center from plane 1:
    lbp::olbp_t< unsigned char, 1, 8 > op;
    auto result = op (plane [0], plane [1]);
    
## OCLBP

The implementation requires a 3-plane image and assumes that incoming frames are
RGB (not BGR). It computes a set of 6 frames out of each incoming frame, where
each of the six is a plain application of Ojala LBP operator of radius 1 and 8
neighbors between pairs of planes: 

- R v. R
- G v. G
- B v. B
- R v. G
- R v. B
- G v. B

The code to apply the operator and display the images can be as simple as this
(using Boost format):

    const auto op = lbp::oclbp_t { };
    
    // Fetch frames ...
    const auto images = op (frame);

    size_t i = 0;
    
    for (const auto& image : images) {
        imshow ((format ("OCLBP, frame %1%") % (i++)).str (), image);
    }

## VARLBP

The implementation requires a single-plane, floating point image for input. It
applies the operator over a neighborhood of 8 pixels, using the Welford online
algorithm for calculating variance. The variance values may go above 1.0, so the
image needs to be normalized before displaying:

    const auto op = lbp::varlbp_t { };

    // Fetch frames ...
    const auto result = op (src);

    // ... and normalize:
    double a, b;
    minMaxLoc (result, &a, &b);
    
    Mat normal = result.convertTo (normal, CV_32FC1, 1/(b - a), -a);
    
    imshow ("VARLBP", normal);

## CSLBP

As in Center Symmetric LBP, see the citation in the header. The implementation
follows the description in chapter 2.2, *Feature Extraction with
Center-Symmetric Local Binary Patterns*. It uses Boost *hana* for some light
meta. The usage is straighforward, create  the operator object:

    const auto op = lbp::cslbp_t { };
    
    // Fetch the source, say, a gray float image:
    Mat src = ...
    
    // Convert the source image with a custom epsilon:
    const auto result = op.operator()< float > (src, 0.05);

The function call operator of the operator object is templatized on the Mat
underlying type. There is no requirement on the input image type other than
being a gray, single-channel.

The destination Mat type is the smallest type that can accommodate the result of
the operator. E.g., a neighborhood of 12, makes 6 comparisons, generating values
in the range [0,2<sup>5</sup>], and it requires at least 8 bits to store it,
i.e., an unsigned char.

Normalize the result into a floating point Mat, range [0,1], and display:

    double a, b;
    minMaxLoc (result, &a, &b);
    
    Mat normal = result.convertTo (normal, CV_32FC1, 1/(b - a), -a);
    imshow ("CS-LBP", normal);

## Parallelization

The implementations are reasonably streamlined within the boundaries of their
idioms. E.g., each operator returns a new frame(s) out of the input image and
when accounting for the cost of that paradigm the code is as fast or faster than
other similar implementations. Optimizing the implementation is something I wish
to do with greater care after some more tinkering with the design.

In the meantime I am using OpenMP in places with nice (expected) results. More
improvements, soon.

## Utilities

There are a bunch of one-line internal utilities in the library that are useful
to me, mostly related to conversion between formats, scaling, etc. Find them in
`src/utils.cpp`. 

I copied Eric Niebler's getline range code and adapted it to fetching frames
from a `cv::VideoCapture` object. It allows for some sweetness:

    cv::VideoCapture cap = ...
    for (const auto& frame : lbp::getframes_from (cap)) {
        // ... use the frame
    }

The code for that is in `include/lbp/frame_range.hpp`, enjoy.

## CMake and Windows

No.
