FROM jekyll/jekyll:4.2.2

WORKDIR /srv/jekyll
COPY . .

RUN bundle config set path /tmp/vendor && bundle install
