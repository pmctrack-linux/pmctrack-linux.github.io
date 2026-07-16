FROM debian:11.6
RUN mkdir /www & apt update && apt -y install vim build-essential ruby2.7 ruby2.7-dev curl wget
RUN gem install bundler github-pages
WORKDIR /tmp
COPY Gemfile Gemfile.lock /
RUN gem install minitest -v '5.13.0' 
RUN bundle install
WORKDIR /www
CMD /bin/bash