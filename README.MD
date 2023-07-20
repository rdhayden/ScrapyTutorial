Create a project first
	Scrapy startproject projectName
	Creates a folder with the projectName which has a scrapy.cfg file
	Also has a subfolder with
		Spiders folders which will be empty initially
		items.py where we clean the data we scrape and assume values extracted to fields
		middlewares.py which has
			A ProjectNameSpiderMiddleware class which orchestrates requests 
			A ProjectNameDownloaderMiddleware class which downloads the HTML
	Then create a spider, from the ProjectName folder
		scrapy genspider nameOfSpider www.sitename.bla/… (note no / at the end and no HTTP)
		This will create a nameOfSpider.py file in the spiders folder (name must be unique). In that file …
		Scrapy creates http:// url by default, change it to https and configure allowed_domains (only domain links spider will scrape and without http:// or https:// in front)
		It has a parse function
			Add code here to process the response and return data
			See below for using the shell to test what xpath returns from a response object before updating this function but the function might look like this
			
			def parse(self, response):
				countries = response.xpath(“//td/a”)
				for country in countries:
					// note when you execute an xpath expression against
			// a select object not a response object, you start the 
			// xpath expression “.//” (dot slash slash) instead of “//”
					name = country.xpath(“.//text()”).get()
					link = country.xpath(“.//@href”).get()

					yield {
						‘title’: title,
						‘countries’ : countries
					}

			You can also follow links and pass their response objects to a new parse function, this is shown below including passing metadata gathered from the first page parsed. Note the first parse function should always be named parse or nothing will work …
			
			    def parse(self, response):
				countries = response.xpath("//td/a")
				for country in countries:
				    name = country.xpath(".//text()").get()
				    link = country.xpath(".//@href").get()

				    # absolute_url = f"https://www.worldometers.info{link}"
				    # absolute_url = response.urljoin(link)

				    # yield scrapy.Request(url=absolute_url)

				    yield response.follow(url=link, callback=self.parse_country, meta={'country_name': name})

			    def parse_country(self, response):
				name = response.request.meta['country_name']
				rows = response.xpath("(//table[@class='table table-striped table-bordered table-hover table-condensed table-list'])[1]/tbody/tr")
				for row in rows:
				    year = row.xpath(".//td[1]/text()").get()
				    population = row.xpath(".//td[2]/strong/text()").get()
				    yield {
					'country_name': name,
					'year': year,
					'population': population
				    }

Then test the crawler
	In the terminal, make sure you’re in the same directory as the scrapy.cfg file (the top level project folder) and type
	Scrapy crawl nameOfSpider
	You should get back the python dictionary with all the elements yielded by the parse function

From ProjectName folder (ensure you have ipython installed) you can launch the scrapy shell which allows you to do things like launch scrapers and see all the scrapy objects created
	scrapy shell 
	You should be able to see the objects 
		scrapy   scrapy module (contains scrapy.Request, scrapy. Selector, etc)
		crawler
		item {}
		settings
		fetch
		shelp (shell help)
		view(response) which opens a response in a browser
	To open a URL you can 
		use fetch(“http://…”)
		Or
		req = scrapy.Request(url=”https://…)
		fetch(req)
	To check the the returned html body, just reference it directly from the response object
		response.body
		Control + U will open it in a browser and display the html markup
	You can also use the view command to view the response object in the browser formatted as expected
		view(response)
		however note that this is deceptive, if you actually open the page and select ctrl + shift + p to open the command pilot, then search for javascript and disable javascript you will see the page as the scraper actually sees it
	After disabling javascript (see previous comment), find specific elements of interest in a chrome browser
		Right click the element of interest in the browser and select inspect
		Make sure the elements tab is select and click ctrl + f
		Type in the xpath for the element (e.g. //h1), check you get the element you want and the count of that element you expect
		Then go back to scrapy shell and type e.g.
			title =  response.xpath(“//h1”)
			You should see that title is now a list with 2 objects in it, a selector object with the xpath property showing the xpath used, and a data property with the result of the xpath. If you just want the data you coud instead type
				title = response.xpath(“//h1”)[1], and to get the text you could then …
				title.get() or if the expression returns multiple things title.getall()

To export data do something like
	scrapy crawl countries -o population_dataset.json
	or
	scrapy crawl countries -o population_dataset.csv
	or
	scrapy crawl countries -o population_dataset.xml