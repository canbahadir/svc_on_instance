# Use an official Node runtime as the parent image
FROM node:6

# Set the working directory in the container to /app
WORKDIR /app

# Copy the current directory contents into the container at /app
ADD . /app

RUN npm install 

# Make the container's port 80 available to the outside world
EXPOSE 8080

# Run app.js using node when the container launches
CMD ["node", "svc_main.js"]