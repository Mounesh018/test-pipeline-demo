## FROM node:latest-alpine as build
 #FROM
- it is used to select a base image for Docker
#node:latest-alpine
it is use for official Node.js image and image version that is latest
alpine means lightweight and Smaller image size that helps us to download and build faster than the normal
#as build
it is Creates a stage with name as build and used for multi stage docker build 
helps copy files from one stage to another

## WORKDIR /app
it is sets the working directory inside container

## ENV PATH /app/node_modules/.bin:$PATH
#ENV
it is sets environment variables inside the container
#PATH
Linux variable that stores executable locations
#/app/node_modules/.bin
Contains local npm binaries

## COPY . ./
#COPY
it copies the files from local machine to container
#.
current local directory
#./
current container working directory

## RUN npm install --force
#RUN
Executes command during image build
#npm install
Installs all dependencies from package.json
#--force
forces installation even if dependency conflicts exist

##RUN npm run build
Executes React production build
#npm run build
Runs build script from package.json

# Production Stage
##FROM nginx:stable-alpine
Starts second stage and uses the official NGINX image
#stable-alpine
Stable nginx version
Lightweight Alpine Linux

##COPY --from=build /app/build /usr/share/nginx/html
#COPY --from=build
Copies files from previous stage build
Source: /app/build
Destination: /usr/share/nginx/html

##COPY nginx/nginx.conf /etc/nginx/conf.d/default.conf
Copies custom nginx configuration
Source: nginx/nginx.conf
Destination: /etc/nginx/conf.d/default.conf
we need this line because the default NGINX configuration is usually not enough for modern frontend applications like React.

##EXPOSE 80
#EXPOSE
Documents container listening port
#80
HTTP default port

##CMD ["nginx", "-g", "daemon off;"]
#CMD
Default command when container starts
#"nginx"
Starts nginx server
# "-g", "daemon off;"
Runs nginx in foreground
#-g
pass global configuration directives directly from command line
#daemon off 
stay in foreground
remain attached to container
So Docker keeps container alive.
#without daemon off
nginx:
starts
moves to background
main process exits
Docker thinks work is finished Then container stops immediately 