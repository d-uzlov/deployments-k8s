
# About this

Files in this folder are part of the work-in-progress performance testing attempt.

There may be some not very clear stuff, because I didn't have time to polish it.

All of the instructions here expect that you have a proper NSM setup, even if it isn't mentioned in the readme file.

# Visualization

If you want to create graphs to visualize the results you can install `fortio`:

[https://github.com/fortio/fortio#installation](https://github.com/fortio/fortio#installation)

Run `fortio server`, then place `.json` files from some `results` folder into the same folder where you launched fortio.

Open [http://localhost:8080/fortio/browse](http://localhost:8080/fortio/browse) and choose the result you are interested in.
