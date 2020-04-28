
## How To

The website uses a remote theme stored at Github. To test it locally perform the following steps

1. Uncomment the following line in `_config.yml`

	#theme: bulma-clean-theme

2. Launch jekyll

	Juans-MBP-2:pmctrack-website jcsaez$ bundle exec jekyll serve

Do not forget to comment back out the aforementioned line in `_config.yml` to ensure the website works correctly on Github Pages.


### In the event of any problem with Jekyll ...

```bash
$ rm Gemfile.lock 
$ bundle install
```

### Where is everything defined?

* Horizontal navigation menu: `data/navigation.yml`
* Body of Downloads page: `data/download.yml`
* Google Analytics Tag:
	- `google_analytics` property in `_config.yml`	

	






