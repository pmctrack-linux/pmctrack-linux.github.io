## Deploy container for web editions

1. Build custom docker image from the root directory of the website

```bash
host:web-pmctrack user$ docker image build -t myjekyll -f docker/Dockerfile . 
```

2. Run container as follows:

```bash
docker container run --rm -it -v $PWD:/www -p 4000:4000 myjekyll bash
```

## Test web locally (from container)

1. Force utilization of local theme, rather than remote theme in *_config.yml.*:

```yaml
# Build settings
theme: bulma-clean-theme
#remote_theme: chrisrhymes/bulma-clean-theme
```

2. Serve website on <http://localhost:4000>

```bash
root@ff2e37cba8dd:/www# bundle exec jekyll serve --host 0.0.0.0
Configuration file: /www/_config.yml
            Source: /www
       Destination: /www/_site
 Incremental build: disabled. Enable with --incremental
      Generating... 
       Jekyll Feed: Generating feed for posts
                    done in 4.128 seconds.
 Auto-regeneration: enabled for '/www'
    Server address: http://0.0.0.0:4000/
  Server running... press ctrl-c to stop.
```


## To publish the changes on github pages ...

* Don't forget to use the remote theme before committing the changes:

```yaml
#theme: bulma-clean-theme
remote_theme: chrisrhymes/bulma-clean-theme
```


