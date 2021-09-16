FROM ruby:latest
LABEL author="Matt Thomas"
LABEL version="2.0.0"

# Install git and bundler
ENV BUNDLER_VERSION 2.2.27
RUN apt-get update && \
    apt-get install --no-install-recommends -y \
    git-all \
    && gem install bundler:2.2.27 \
    && bundle config --global silence_root_warning 1

# This is our entrypoint to our custom scripts
# more about that in a sec
COPY entrypoint.sh /

# Use the entrypoint.sh file as the container entrypoint
# when Github executes our Docker container
ENTRYPOINT ["sh", "/entrypoint.sh"]