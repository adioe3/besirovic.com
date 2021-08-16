---
layout: post
title: "Magic numbers and technical debt"
date: 2015-07-24
---

Back in 2012 we had an inherited (let's call it legacy) project which was tossed around for quite some time between a number of developer teams.
The original developers were German so we had half of the database tables in German.
Then at some point they had an international team which spoke English so the other half of the database tables are in English.
To this day, to collect data for a single order you have to do joins between the `bestellung` and `order` tables.

The project was a web-to-print platform, meaning that you'd upload a photo of your dog dressed up in silly clothes and get a nice poster out of it.
We also did canvas prints and this is where the fun begins.

## Greater than 6663410

After their initial success, the client opened up a shop in Australia and that meant we needed to set up a fresh new installation of the platform for the Aussies, which we did. As soon as the first canvas orders started coming in we noticed every canvas had a 2.1cm white edge around and digging into the code I came upon this constant:

```php
const NONE_WHITE_BORDER_ORDER_ID = 6663410;
```

No comment, no nothing, but later in the code you'd see this chunk:

```php
if ($this->getId() && $this->getId() < static::NONE_WHITE_BORDER_ORDER_ID) {
    $additionalWhiteBorder = static::ADDITIONAL_WHITE_BORDER_CM;
}
```

Obviously, as soon as the ID hits that magic number we stop adding the white border.
So, after a bunch of digging through ancient SVN logs and JIRA tickets I found the answer as to what this magic number really does and it has to do a bit with the client's history.

Storytime!

Back in the day, this client was a startup, meaning they could not afford expensive printing and cutting machines so they simply had the cheapest available plotters around.
Since the plotters were not able to account for precise canvas positioning and material stretching, they would print out an additional white border and then cut them by hand.

Time went by and profits rose high enough for them to invest into high-end precise positioning, cutting and printing machines which meant we didn't need to add the white border anymore as these machines were made with material stretching in mind.

So, what the jolly legacy developer did was simply taking the last order ID at that time and smacked it into this constant thus any new orders (the ID's an `AUTO_INCREMENT`) would be without the white border.
This was obviously less then ideal if you've just set up a new platform and your order ID is 1.

## Divide by 2000

There is a very real problem with web-to-print platforms which is that you inevitably run into calculation precision problems.
There's always some photo editor which works on a scaled down version of the original image (in our case a thumbnail) where you can crop and add effects to the image.
Since even in 2015 high-DPI monitors are a thing of science fiction you can imagine how much fun it is to convert a cropped area from a 300x200px thumbnail to a 40x30cm poster.
There'll always be some loss and we settled on an approved error margin to the third decimal (i.e. calculated 39.991cm was acceptable for print).

To make life easier and not print a single pixel to a 2x2cm area we had methods to calculate the quality of the uploaded image and then reject any image we'd find sub-optimal. Digging around the code we eventually found the quality calculation method.
This one had a comment:

```php
/*
 * Nobody knows why we divide by 2000. 2000 is not even a measurement divisor
 */
$quality = round($dpi * (1 + ($printDimension / 2000)));
return $quality;
```

I've spent north of three months trying to understand that. The conclusions I drew were:

- `$dpi` is in fact the PPI (pixels per inch) which is different from actual DPI: PPI is a measure for displays, DPI is a measure for printers; as a side note it really is the PPmm (pixel per millimeter)
- `$printDimension` there is a pixel value
- everything after the $dpi basically increases the calculated quality by a percentage which revolves around how close the print dimension is to 2000px

I've scoured documentation and instruction manuals for various used plotters and printers and could not find a good reason why we'd consider any image over 2000px as a greater quality image.
Essentially, if your uploaded image is 2000px or greater, the calculated qualification (how good the image is for print) would be reported as optimal.

To this day, the 2000px number remains a mistery.

Time went by and I was assigned to other projects but even today this code is happily resting in the code base and what I've learned from it is that you should always try and document the decisions made during the project.
Some days I still revisit this code and try again to figure out what the 2000px line is for but, regardless of how hard you try to write clean code, only a comment can document a business decision.

