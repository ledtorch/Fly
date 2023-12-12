# Project Overview
This project, named "Whom Are Flying With You," aims to visualize your travel routes, showing where you're flying to or from. It also provides data about how many people are flying with you. 

Flying in the sky for long distances can be a lonely experience. You can only see the passengers and crew on the vessel. I want to alleviate this loneliness. Surprisingly, few people know that approximately a million people are in the sky at any given time.

Briefly, it's a project that visualizes your journey with data.

----

### Keywords for searching resources
- Three.js Globe Sphere Dots

### References
- Stripe Globe Visualization: [Stripe Globe](https://stripe.com/blog/globe)
- Real Time Flight Spotter: [Flight Spotter GitHub](https://github.com/janhartmann/flight-spotter)
- Globe.GL: [Globe.GL](https://globe.gl/)

### GPT Project Configuration Description
This project is a visualization tool developed with Vue3, Vite, and Three.js. It features an interactive globe where dots represent geographic locations, and dynamic flight routes are depicted based on user input.

#### Features
- **Interactive Globe Rendering**: A sphere representing the Earth, with dots indicating landmasses.
- **Flight Route Visualization**: Users can input an airline code (e.g., DTK253), and the app will draw a parabolic line between the departure and arrival airports to visualize the flight path.
- **Airborne Traffic Calculation**: The app calculates and displays the number of airplanes and passengers airborne at the time of a specific flight's landing.
- **Concurrent Flight Paths**: It also depicts the flight paths of all airplanes that were in the air during the selected flight's duration.

#### Example
When a user inputs 'DTK253', the app will display the number of aircraft and passengers in the air when DTK253 lands. For example, if 30 airplanes with a total of 30,000 passengers are airborne at that time, the app will show this data and trace the paths of these 30 flights.

### Project Structure
.
├── index.html
├── package.json
├── public
│ ├── Button
│ │ └── Icon
│ └── Image
│ ├── Nebula.jpg
│ └── map.png
├── src
│ ├── App.vue
│ ├── assets
│ │ └── vue.svg
│ ├── components
│ │ ├── Button
│ │ └── TheSphere.vue
│ ├── main.js
│ └── style.css
├── vite.config.js
└── yarn.lock


### Package Installation
```bash
# Env dependencies
yarn add sass --dev
yarn add postcss autoprefixer --dev

# Feature Dependencies
yarn add axios
```

#### Sample flight code: 
> NH110, B61724, NH110, AA142, DL1538, DL4835, B6881, B62201, B6219, AA4565