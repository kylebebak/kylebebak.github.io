---
layout: page
title: Code
permalink: /code
custom_css: page
---

## Python

- __questionnaire__ &mdash; a Python package that prompts users to answer a series of questions using a terminal GUI, and returns the answers. questionnaire makes it trivial to ask different questions based on previous answers, and it allows users to go back and answer questions again. It works with Python 2 and 3.

  The package currently has 3 core prompters that can ask __multiple choice/single option__, __multiple choice/multiple option__, and __raw input__ questions. Extending the package is as simple as writing new prompters. [Check it out here](https://github.com/kylebebak/questionnaire).


## JavaScript

- __react-dropzone-uploader__ &mdash; [customizable HTML5 file dropzone and uploader for React](https://github.com/fortana-co/react-dropzone-uploader), with progress indicators, upload cancellation and restart, and minimal dependencies.


## Sublime Text

- __Requester__ &mdash; [A simple, powerful HTTP client](https://github.com/kylebebak/Requester) built on top of [Requests](http://docs.python-requests.org/en/master/). Like having Postman in your text editor.

- __open-url__ &mdash; [A plugin](https://github.com/noahcoad/open-url) to quickly open files, folders, web URLs or other URLs from anywhere in Sublime Text.


## Other

- __Notes__ &mdash; Notes for hackers. Notes provides a minimal syntax for quick access, editing capability, and search functionality to all notes under a directory of your choosing. notes).


- __Location History__ &mdash; Unless you've disabled location services on your phone, Google (or Apple or Microsoft) is probably [tracking your location](http://www.howtogeek.com/195647/googles-location-history-is-still-recording-your-every-move/), in Google's case once a minute and accurate to 5 or 10 meters. There used to be an API for accessing this data, but now the best you can do is download your raw location history for a range of dates as KML or JSON via Google Takeout. I did this, ran a clustering algorithm on the data to create visit, location and trip instances, and put them in a database.

  When run on about 18 months of my location data, this process turned up 4400 visits (clusters of points where I'd been stationary somewhere for more than 6 minutes) that correspond to 750 unique locations. Trips are the sequences of moving points that occur between visits. The data was a lot of fun to explore. Probed with simple queries it can answer interesting aggregate questions, like where are the top five places I spend time on Saturdays, or, over a period of 6 months, at what time on average did I leave work on each of the different weekdays.

  I built a front end for this data using the Google Maps API and some JS plugins. The data doesn't reveal anything that makes me uncomfortable, [so I made it available here](https://dronfelipe.onrender.com/location_history).

- __Tortas__ &mdash; Pon tu propio changarro de tortas. Genera un [menú](https://dronfelipe.onrender.com/tortas) digno de los mejores puestos: sumamente variado, internacional y sin sentido alguno.
