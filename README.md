# Live Project 
2/15-26/2021
## Introduction
This project comprised a two week sprint where I was tasked with building a full scale MVT Web App for tracking something using a database. The three main goals were to create CRUD functionality for the database using Django models, scrape information from another site using beautiful soup, and access an API to display JSON responses.
## Stories
The app I decided to create was one that could be used to track upcoming concerts that the user might be interested in. I am a classical musician, so the database tables and other stories all centered around classical music concerts. 
### Create the Models
The models are the classes that Django uses to migrate information between the browser and the database. I had four models: 
1. Pieces of Music
2. Orchestra Information
3. Conductors
4. Events (concerts)

Tables 1-3 were linked to the Events table using both foreign keys and Django's builtin ManyToManyField for models. The ManyToManyField allows an entry to be found on many tables, and for each table to have many of that type of entry. In this case, a piece of music can be found on many different concerts, and each concert can have many different pieces programmed. 
```python
# had to be last because it references 
# all of the other tables with foreign keys
class Concert(models.Model): 
    event_name = models.CharField(max_length=100)
    orchestra = models.ForeignKey(Orchestra, on_delete=models.CASCADE,)
    pieces = models.ManyToManyField(Piece)
    conductor = models.ForeignKey(Conductor, on_delete=models.CASCADE,)
    date = models.DateField()

    concerts = models.Manager()

    def __str__(self):
        return self.event_name
```
When rendered as a form, this model makes a nice set of entries, with dropdowns for tables linked by foreign keys and a milti-click box for the ManyToManyField. 
![Concert Model](/img/add_event_snip.jpg)

Understanding how the ManyToManyField works became essential when I tried to display the database results on a template. Initially I was unable to return the pieces programmed on each concert. I discoverd that the ManyToManyField creates another table in the database, holding foreign keys for all of the pieces and concerts on which they are programmed. I had to use a "through" statement in my view function to access them:
```python
def details(request, pk):
    event = get_object_or_404(Concert, pk=pk)
    all_event_pieces = Concert.pieces.through.objects.all()
    all_pieces = Piece.pieces.all()
    return render(request, 'UpcomingConcertsApp/details.html',
                  {'event': event, 'all_event_pieces': all_event_pieces,
                   'all_pieces': all_pieces})
```
This development also took some tricky looping on the template to find which pieces go with each concert and display them. the end result was a details page that displayed the concert information and program: 
![Details Page](/img/details.jpg)

### Web Scraping
Web Scraping lends itself to my project because many orchestras publish their concert season on their website. They don't have an API and there are no aggregator sites where their season can be found. I decided to scrape the upcoming livestream concerts from the Berlin Philharmonic. Since our framework was Django, I used beautiful soup to accomplish this. 
```python
def berlin_scrape(request):
    page = requests.get('https://www.digitalconcerthall.com/en/live')
    soup = BeautifulSoup(page.content, 'html.parser')
    concert_list = soup.find(id="results")
    list_of_concerts = concert_list.find_all(class_="live")
    concerts = []

    for concert in list_of_concerts:
        title = concert.find(class_='concertTitle').get_text()

        artists = concert.find(class_='head')
        artists_h2 = list(artists.children)[0]
        artist_entries = [person.get_text() for i, person in enumerate(artists_h2) if i % 2 == 0]
        # The following accomplishes the same purpose as the above code. Neither is perfect,
        # as the below has nested loops. Above, it grabs the h2 element below the class called "head"
        # and then loops through the list skipping every other entry, which happens to be a <br> tag.
        # artist_entries = []
        # artist_list = concert.find_all('div', class_='head')
        # for artists in artist_list:
        #     artist = artists.select('h2 span')
        #     for i in artist:
        #         artist_entries.append(i.get_text())

        stars = concert.find_all(class_='stars')
        star_entries = [person.get_text() for person in stars]

        work_list = []
        work_html = concert.find_all(class_='work')
        for works in work_html:
            work = works.select('h3')
            for w in work:
                work_list.append(w.get_text())

        a_tag = concert.find_all('a')
        link = a_tag[0].get('href')

        concert_info = {'title': title, 'artists': artist_entries,
                        'stars': star_entries, 'work_list': work_list, 'link': link}
        concerts.append(concert_info)

    return render(request, 'UpcomingConcertsApp/scraped_concerts.html', {'concerts': concerts})
```

The trickiest aspects of scraping came in when multiple tags were used per line. Sometimes text would be wrapped in *span* tags, followed by *br* tags. Because of the way beautiful soup works these breaks were difficult to sift out. You will find a list comprehension for **artist_entries** that takes care of this problem. There is another way to do it which I left for future users in case the current way stops working. In the end it provided a nice set of concerts to view on the template:
![Web Scraping Page](/img/scraped_concerts.jpg)

### Working with an API
I had a great experience working with Open Opus, a classical music API that gives access to a great database of composers, pieces, and informationa about the eras in which they lived. I used the API to create a search page for the user, in which they could search the linked database for information about specific composers or works they have composed. I also used it to display a short list of popular composers on the side of the page. The final touch was error handling, which I did in simple python try/except statements. My view function is below, which renders the page. 
```python
def open_opus(request):
    composer_data = {}
    work_data = {}
    success = ""
    pc_response = requests.get('https://api.openopus.org/composer/list/pop.json')
    pc_result = pc_response.json()
    popular_composers = pc_result['composers']
    count = 0
    p_composers = []
    for composer in popular_composers:
        if count < 5:
            p_composers.append(composer)
            count += 1

    if 'find_composer' in request.GET:
        try:
            search_criteria = request.GET['composer']
            url = 'https://api.openopus.org/composer/list/search/' + search_criteria + '.json'
            response = requests.get(url)
            search_result = response.json()
            composer_data = search_result['composers']
        except KeyError:
            success = "That didn't return a valid response. Try a more general search"
    elif 'find_works' in request.GET:
        try:
            works = request.GET['works']
            composer = request.GET['work_composer']
            url = 'https://api.openopus.org/work/list/composer/' + composer + \
                  '/genre/all/search/' + works + '.json'
            response = requests.get(url)
            search_result = response.json()
            composer_data = [search_result['composer']]
            work_data = search_result['works']
        except KeyError:
            success = "That didn't return a valid response.\n" \
                      "Did you look up the composer first and enter their id?\n" \
                      "If so, try more general search criteria for the works, or " \
                      "leave it empty. "

    elif 'list_by_period' in request.GET:
        composers = request.GET['period']
        url = 'https://api.openopus.org/composer/list/epoch/' + composers + '.json'
        response = requests.get(url)
        search_result = response.json()
        composer_data = search_result['composers']

    return render(request, 'UpcomingConcertsApp/api_classical_music.html',
                  {'composer_data': composer_data, 'work_data': work_data,
                   'popular_composers': p_composers, 'success': success})
```
![API page](/img/api_json.jpg)