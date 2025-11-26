FROM nginx:latest
LABEL "Project"="Vprofile" 
LABEL "Maintainer"="samir"

RUN rm -rf /etc/nginx/conf.d/default.conf
COPY nginvproapp.conf /etc/nginx/conf.d/vproapp.conf