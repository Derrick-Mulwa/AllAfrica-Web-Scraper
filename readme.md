# AllAfrica Web Scraper

**AllAfrica Web Scraper** is a professional Python-based tool designed to extract and store news articles from [AllAfrica](https://allafrica.com). This project leverages Selenium for dynamic text and image data web scraping, concurrent processing for efficiency, and SQLite for structured data storage. It is ideal for collecting news data, including article metadata, content, and images, in a scalable and robust manner. 

## Table of Contents
1. [Project Overview](#project-overview)
2. [Features](#features)
3. [Project Structure](#project-structure)
4. [Workflow](#workflow)
5. [Advantages](#advantages)
6. [Installation](#installation)
7. [Usage](#usage)
8. [License](#license)

## Project Overview
The AllAfrica Web Scraper systematically extracts news articles from AllAfrica's news pages, capturing metadata (e.g., headline, author, publication date), content (e.g., story text, embedded links), and images. Data is stored in a SQLite database for easy querying, with support for concurrent scraping and proxy rotation to handle large-scale tasks. The project showcases expertise in:
- **Web Scraping**: Handling dynamic content with Selenium.
- **Concurrent Processing**: Using `concurrent.futures` for parallel execution.
- **Database Management**: Storing structured data with SQLite and Pandas.
- **Error Handling**: Implementing retries and logging for reliability.

## Features
- Scrapes article metadata, content, and images from AllAfrica.
- Supports concurrent scraping with multithreading.
- Rotates proxies to avoid IP bans.
- Downloads images in JPEG or GIF format.
- Stores data in a SQLite database with a structured schema.
- Tracks progress with a log file to resume scraping after interruptions.
- Maps country names to ISO codes for standardized data.

## Project Structure
The project is organized for clarity and maintainability:

```python
AllAfricaWebScraper/
├── images/                   # Directory for downloaded images
├── database.db               # SQLite database for article data
├── log.txt                   # Log file for tracking progress
├── proxies.txt               # List of proxies for rotation
├── scraper.py                # Main scraping script
└── README.md                 # Project documentation
```

### Key Files
- **`scraper.py`**: Core logic for scraping, processing, and storing data.
- **`database.db`**: Stores article data in a `data` table.
- **`log.txt`**: Tracks `attachments_number`, `last_scrapped_page`, `last_scrapped_link`, `most_recent_scrapped_link`, and `pk_i_id`.
- **`proxies.txt`**: Contains proxies for rotation.
- **`images/`**: Stores downloaded images with sequential naming.

## Workflow
The AllAfrica Web Scraper operates in a streamlined, fault-tolerant manner. Below is the workflow with key code snippets:

1. **Initialization**:
   - Checks for `database.db` and initializes it if absent.
   - Loads proxies from `proxies.txt` and sets initial variables from `log.txt`.
   ```python
   if not os.path.isfile('database.db'):
       print('First time scraping. Initializing the scrapper.')
       attachments_number = 1
       last_scrapped_page = 1
       pk_i_id = 1
       os.mkdir('images')
   ```

2. **Scraping New Articles**:
   - Fetches article URLs from `https://allafrica.com/latest/`.
   - Identifies new articles by comparing against `most_recent_scrapped_link`.
   - Scrapes articles concurrently.
   ```python
   while caught_up_on_new_posts == False:
       links = get_page_article_urls(current_page)
       if most_recent_scrapped_link in links:
           new_article_urls = links[:links.index(most_recent_scrapped_link)]
           new_article_links += new_article_urls
           caught_up_on_new_posts = True
       else:
           new_article_links += links
           current_page += 1
   ```

3. **Scraping Older Articles**:
   - Resumes from `last_scrapped_page` and `last_scrapped_link`.
   - Processes articles in batches, moving to subsequent pages.
   ```python
   while found_last_scraped_link == False:
       links = get_page_article_urls(last_scrapped_page)
       if last_scrapped_link in links:
           article_urls = links[links.index(last_scrapped_link)+1:]
           found_last_scraped_link = True
       else:
           last_scrapped_page += 1
   ```

4. **Article Scraping (`scrape_page`)**:
   - Uses headless Selenium to extract headline, author, publication date, story text, embedded links, images, and tags.
   - Maps country names to ISO codes.
   ```python
   def scrape_page(url):
       options = Options()
       options.headless = True
       driver = webdriver.Chrome(options=options)
       driver.get(url)
       time.sleep(5)
       headline = driver.find_element(By.CLASS_NAME, 'headline').text
       story_body = driver.find_element(By.CLASS_NAME, 'story-body')
       story_text = '\n\n'.join([para.text for para in story_body.find_elements(By.TAG_NAME, 'p') if not para.text.startswith('RELATED:')])
       driver.close()
       return [headline, article_image_names, ..., story_text, ...]
   ```

5. **Image Downloading (`download_image_func`)**:
   - Downloads images with retry logic, saving as JPEG or GIF.
   ```python
   def download_image_func(image_url):
       retries = Retry(total=3, backoff_factor=1, status_forcelist=[500, 502, 503, 504])
       session = requests.Session()
       session.mount("https://", HTTPAdapter(max_retries=retries))
       response = session.get(image_url, proxies=proxy_dict, timeout=15)
       if response.status_code == 200:
           image = Image.open(io.BytesIO(response.content))
           with open(f'images/attachment_{attachments_number}.jpg', 'wb') as f:
               image.save(f, 'JPEG')
   ```

6. **Data Storage (`add_to_db`)**:
   - Stores data in `database.db` using Pandas and SQLite.
   - Increments `pk_i_id` and logs progress.
   ```python
   def add_to_db(extracted_data_list):
       df = pd.DataFrame(columns=columns_article_data_df, data=[[pk_i_id]+extracted_data_list])
       with lock:
           conn = sqlite3.connect('database.db')
           df.to_sql('data', conn, if_exists='append', index=False)
           conn.close()
           pk_i_id += 1
           log_progress()
   ```

7. **Concurrent Execution**:
   - Uses a thread pool with 4 workers for parallel scraping, ensuring thread safety with `threading.Lock`.
   ```python
   with concurrent.futures.ThreadPoolExecutor(max_workers=4) as executor:
       future_to_url = {executor.submit(scrape_and_save_older_articles, url): url for url in article_urls}
   ```

8. **Progress Logging (`log_progress`)**:
   - Updates `log.txt` with tracking variables.
   ```python
   def log_progress():
       log_txt = f'attachments_number = {attachments_number}\nlast_scrapped_page = {last_scrapped_page}\n...'
       with open('log.txt', 'w') as f:
           f.write(log_txt)
   ```

## Advantages
- **Scalability**: Multithreading and proxy rotation enable high-volume scraping.
- **Reliability**: Retries, exception handling, and progress logging ensure uninterrupted operation.
- **Data Organization**: Structured SQLite storage facilitates querying and analysis.
- **Efficiency**: Headless Selenium and concurrent processing minimize runtime.
- **Flexibility**: Modular design allows adaptation to other websites.

## Installation
### Prerequisites
- Python 3.8+
- Dependencies: `selenium`, `requests`, `pandas`, `pillow`
- Chrome WebDriver (compatible with your Chrome version)

### Steps
1. **Clone the Repository**

2. **Install Dependencies**:
   ```plaintext
   pip install selenium requests pandas pillow
   ```

3. **Set Up Chrome WebDriver**:
   - Download from [here](https://sites.google.com/chromium.org/driver/).
   - Add to system PATH or project directory.

4. **Prepare Proxies** (Optional):
   - Create `proxies.txt` with proxies (e.g., `http://proxy:port`).
   - Comment out proxy code if not needed.

5. **Create Image Directory**:
   ```plaintext
   mkdir images
   ```

6. **Run the Scraper**:
   ```plaintext
   python scraper.py
   ```

## Usage
The scraper runs continuously, handling two modes:
1. **Initial Run**:
   - Initializes `database.db` and starts from page 1.
   - Output: `First time scraping. Initializing the scrapper.`
2. **Resume Run**:
   - Resumes from `last_scrapped_page` and `last_scrapped_link`, scraping new and older articles.
   - Output: `Scraping 5 new articles found. Resuming scraping where we left off.`

### Querying Data
Query the database with SQLite or Pandas:
```python
import sqlite3
import pandas as pd
conn = sqlite3.connect('database.db')
df = pd.read_sql_query("SELECT s_title, fk_author_name, fk_source_link FROM data", conn)
print(df)
conn.close()
```

## Conclusion
 The AllAfrica Web Scraper is a robust, scalable solution for extracting and organizing news data from AllAfrica. With its concurrent processing, proxy rotation, and reliable data storage, it showcases advanced web scraping techniques and programming expertise, making it a valuable tool for data collection and analysis.
