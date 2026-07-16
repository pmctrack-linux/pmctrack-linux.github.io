# Dockerfile
FROM ruby:3.3.4-bookworm

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
       build-essential \
       git \
       nodejs \
       locales \
    && rm -rf /var/lib/apt/lists/*

ENV LANG=C.UTF-8
ENV JEKYLL_ENV=production

## Test jekill first
WORKDIR /tmp
COPY Gemfile Gemfile.lock /
RUN gem install minitest  
RUN bundle install

WORKDIR /www


CMD /bin/bash

#EXPOSE 4000

#CMD ["bundle", "exec", "jekyll", "serve", \
#     "--host", "0.0.0.0", \
#     "--port", "4000", \
#     "--trace"]