FROM ruby:2.2
RUN apt-get update -qq && \
    apt-get install -y --no-install-recommends build-essential &&\
    rm -rf /var/lib/apt/lists/*

ENV APP_HOME /app
RUN mkdir $APP_HOME
WORKDIR $APP_HOME

COPY Gemfile* $APP_HOME/
RUN bundle install
COPY . $APP_HOME

ENV COMMENT_DATABASE_HOST=comment_db \
    COMMENT_DATABASE=comments

CMD ["puma"]
