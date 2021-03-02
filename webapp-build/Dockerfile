FROM ruby:2.7

COPY service.rb Gemfile config.ru /app/

WORKDIR /app

RUN bundle install

CMD ["rackup", "--port", "8080", "--env", "production"]
