FROM httpd:2.4-alpine
RUN rm -rf /usr/local/apache2/htdocs/*
COPY dist/ /usr/local/apache2/htdocs/
EXPOSE 80
CMD ["httpd-foreground"]
