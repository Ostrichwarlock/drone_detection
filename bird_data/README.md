# Bird image benchmark data

This local benchmark contains 200 validation photos and 371 individual bird boxes from Open Images. The sample uses the Open Images `Bird` category (`/m/015p6`) and excludes images with depicted birds, grouped bird boxes, listed image rotation or a positive `Drone` image label. Absence of that label gives limited information about objects elsewhere in a photo. [Open Images V7 annotations](https://storage.googleapis.com/openimages/web/download_v7.html)

The sample gives smaller birds more coverage. In 80 photos, the largest labeled bird box covers at most 10% of the image. The other 120 photos have larger bird boxes. Each group was sampled from sorted validation IDs with Python's `random.Random(883)`. These are general bird photos and may include perched birds, so the results describe this sample of still images.

- `images/` holds the local JPEGs from the CVDF validation mirror. [CVDF Open Images mirror](https://github.com/cvdfoundation/open-images-dataset)
- `manifest.csv` records the exact image URLs, source pages, photographers, licenses and largest bird-box area per photo.
- `bird_boxes.csv` gives the image filename and normalized left, top, right and bottom coordinates for each bird box.
