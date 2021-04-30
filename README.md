# Movies-ETL

## Table of Contents
* [Overview](https://github.com/rkaysen63/Movies-ETL/blob/master/README.md#overview)
* [Resources](https://github.com/rkaysen63/Movies-ETL/blob/master/README.md#resources)
* [Results](https://github.com/rkaysen63/Movies-ETL/blob/master/README.md#results)

## Overview:

The purpose of this project is to prepare reusable code to clean, join and load movie data from Wikipedia and Kaggle into a database for use in a hackathon. The final code  is reusable, thereby allowing the user to take in new data and add to existing tables.

## Resources

* Data Sources:
  * movies_metadata.csv
  * ratings.csv
  * wikipedia-movies.json
* Software: Python 3.7.9 in Jupyter Notebook interface
* Database Management System:  PostgreSQL
* Development Platform:  pgAdmin
* Lesson Plan: UTA-VIRT-DATA-PT-02-2021-U-B-TTH, Module 8 Challenge

## Results:

* `def clean_movie(movie):` is a function that will   
  * combine alternate titles located in the various language columns into one column called 'alt_titles"
  * remove the nearly empty language columns
  * and merge similar column names.
    
          def clean_movie(movie):
          movie = dict(movie) #create a non-destructive copy
          alt_titles = {}
          # combine alternate titles into one list

          for key in ['Also known as','Arabic','Cantonese','Chinese','French',
                      'Hangul','Hebrew','Hepburn','Japanese','Literally',
                      'Mandarin','McCune–Reischauer','Original title','Polish',
                      'Revised Romanization','Romanized','Russian',
                      'Simplified','Traditional','Yiddish']:  

              if key in movie:
                  alt_titles[key] = movie[key]
                  movie.pop(key)
          if len(alt_titles) > 0:
              print(alt_titles)
              movie['alt_titles'] = alt_titles

          # merge column names
          def change_column_name(old_name, new_name):
              if old_name in movie:
                  movie[new_name] = movie.pop(old_name)
          change_column_name('Adaptation by', 'Writer(s)')
          change_column_name('Country of origin', 'Country')
          change_column_name('Directed by', 'Director')
          change_column_name('Distributed by', 'Distributor')
          change_column_name('Edited by', 'Editor(s)')
          change_column_name('Length', 'Running time')
          change_column_name('Original release', 'Release date')
          change_column_name('Music by', 'Composer(s)')
          change_column_name('Produced by', 'Producer(s)')
          change_column_name('Producer', 'Producer(s)')
          change_column_name('Productioncompanies ', 'Production company(s)')
          change_column_name('Productioncompany ', 'Production company(s)')
          change_column_name('Released', 'Release date')
          change_column_name('Release Date', 'Release date')
          change_column_name('Screen story by', 'Writer(s)')
          change_column_name('Screenplay by', 'Writer(s)')
          change_column_name('Story by', 'Writer(s)')
          change_column_name('Theme music composer', 'Composer(s)')
          change_column_name('Written by', 'Writer(s)')

          return movie
    
* `def extract_transform_load (wiki_data, kaggle_data, ratings_data):` is a function that will do as the name implies: extract the messy data, transform it and then load it into two tables set up in PostgreSQL.  The `def extract_transform_load (wiki_data, kaggle_data, ratings_data):` is a function that calls the `def clean_movie(movie):` function and uses several other named and unnamed functions, i.e. lambda functions, and list comprehensions to do the work.  The function extracts, or reads, the Kaggle Metadata and ratings data from csv files using the Pandas module while the Wikipedia data is extracted from a JSON file using the JSON module.  The transformation of the data includes, but is not limited to filtering out rows not required, dropping duplicates, dropping null columns, parsing the monetary strings in various forms to convert them to numerical values, converting timestamps to date format, filling in missing values with zero, merging DataFrames, merging similar columns, dropping columns, renaming columns.  Then a connection to postgreSQL is created to upload the clean movie data into an existing table, dropping the ratings table from the database and uploading new ratings data into the database to replace it and running a timer for elapsed time to load the ratings data into the database in bins.

      def extract_transform_load (wiki_data, kaggle_data, ratings_data):
          # Read in the kaggle metadata and MovieLens ratings CSV files as Pandas DataFrames.
          kaggle_metadata = pd.read_csv(kaggle_data, low_memory=False)
          ratings = pd.read_csv(ratings_data)
      
          with open(wiki_data, mode='r') as file:
              wiki_movies_raw = json.load(file)    
    
          # List comprehension to filter out TV shows.
          wiki_movies = [movie for movie in wiki_movies_raw
                     if ('Director' in movie or 'Directed by' in movie)
                         and 'imdb_link' in movie
                         and 'No. of episodes' not in movie]    

          # List comprehension to iterate through the cleaned wiki movies list and call the clean_movie function on each movie.
          clean_movies = [clean_movie(movie) for movie in wiki_movies]    

          # Read in the cleaned movies list as a DataFrame.
          wiki_movies_df = pd.DataFrame(clean_movies)

          # Try-except block to catch errors while extracting the IMDb ID using a regular expression string and
          #  dropping any imdb_id duplicates. 
          try: wiki_movies_df['imdb_id'] = wiki_movies_df['imdb_link'].str.extract(r'(tt\d{7})')
          except Exception as e:
              print(e)
        
          # Drop duplicates of IMDb.  (inplace=true will change data frame.)
          wiki_movies_df.drop_duplicates(subset='imdb_id', inplace=True)
        
          #  List comprehension to keep the columns that don't have null values from the wiki_movies_df DataFrame. 
          wiki_columns_to_keep = [column for column in wiki_movies_df.columns if wiki_movies_df[column].isnull().sum() < len(wiki_movies_df) * 0.9]
          wiki_movies_df = wiki_movies_df[wiki_columns_to_keep]
    
          # Create a variable that will hold the non-null values from the “Box office” column.
          box_office = wiki_movies_df['Box office'].dropna()
    
          # Convert the box office data to string values.
          box_office = box_office.apply(lambda x: ' '.join(x) if type(x) == list else x)
    
          # Regular expression to match the six elements of "form_one" of the box office data.
          # First form, "$123.4 million" (or billion) 
          form_one = r'\$\s*\d+\.?\d*\s*[mb]illi?on'

          # Regular expression to match the three elements of "form_two" of the box office data.
          # Second form, e.g. "$123,456,789"
          # \s* - 0 or more spaces after "$"
          # [,\.] to allow comma or period as thousands separator, period requires escape backslash
          # add negative lookahead group for "million" or "billion" after number and rejects match
          form_two = r'\$\s*\d{1,3}(?:[,\.]\d{3})+(?!\s*[mb]illion)'    

          # Function to parse_dollars.
          def parse_dollars(s):
              # if s is not a string, return NaN
              if type(s) != str:
                  return np.nan

              # if input is of the form $###.# million
              if re.match(r'\$\s*\d+\.?\d*\s*milli?on', s, flags=re.IGNORECASE):

                  # remove dollar sign and " million"
                  s = re.sub('\$|\s|[a-zA-Z]','', s)

                  # convert to float and multiply by a million
                  value = float(s) * 10**6

                  # return value
                  return value

              # if input is of the form $###.# billion
              elif re.match(r'\$\s*\d+\.?\d*\s*billi?on', s, flags=re.IGNORECASE):
    
                  # remove dollar sign and " billion"
                  s = re.sub('\$|\s|[a-zA-Z]','', s)

                  # convert to float and multiply by a billion
                  value = float(s) * 10**9

                  return value

              # if input is of the form $###,###,###
              elif re.match(r'\$\s*\d{1,3}(?:[,\.]\d{3})+(?!\s[mb]illion)', s, flags=re.IGNORECASE):

                  # remove dollar sign and commas
                  s = re.sub('\$|,','', s)

                  # convert to float
                  value = float(s)

                  return value

              # otherwise, return NaN
              else:
                  return np.nan    
      
          # Clean the box office column in the wiki_movies_df DataFrame.
          wiki_movies_df['box_office'] = box_office.str.extract(f'({form_one}|{form_two})', flags=re.IGNORECASE)[0].apply(parse_dollars)

          # Drop 'Box Office' from Wiki Movies because there is a clean 'box_office'
          wiki_movies_df.drop('Box office', axis=1, inplace=True)
    
          #---------------------------------------------------------------------------------------------
          # Clean the budget column in the wiki_movies_df DataFrame.
          # Create a variable that will hold the non-null values from the “Budget” column and convert to string.
          budget = wiki_movies_df['Budget'].dropna().apply(lambda x: ' '.join(x) if type(x) == list else x)
    
          # Search for any string that has a citation reference, [#], and remove it.
          budget = budget.str.replace(r'\$.*[-—–](?![a-z])', '$', regex=True)
    
          # Extract the values using str.extract and apply the function parse_dollars
          wiki_movies_df['budget'] = budget.str.extract(f'({form_one}|{form_two})', flags=re.IGNORECASE)[0].apply(parse_dollars)

          # Drop Budget from Wiki Movies because it is no longer required
          wiki_movies_df.drop('Budget', axis=1, inplace=True)
    
          #----------------------------------------------------------------------------------------------
          # Clean the release date column in the wiki_movies_df DataFrame.
    
          # Create a variable that will hold the non-null values from the “Releasae date” column and convert to string.
          release_date = wiki_movies_df['Release date'].dropna().apply(lambda x: ' '.join(x) if type(x) == list else x)
    
          # Create date form one.  month, dd, yyyy format
          date_form_one = date_form_one = r'(?:January|February|March|April|May|June|July|August|September|October|November|December)\s[123]\d,\s\d{4}'

          # Create date form two.  yyyy.mm.dd format
          date_form_two = r'\d{4}.[01]\d.[123]\d'

          # Create date form three.  month yyyy format
          date_form_three = r'(?:January|February|March|April|May|June|July|August|September|October|November|December)\s\d{4}'

          # Create date form four.  yyyy
          date_form_four = r'\d{4}'

          # Extract dates
          release_date.str.extract(f'({date_form_one}|{date_form_two}|{date_form_three}|{date_form_four})', flags=re.IGNORECASE)

          # Parse dates using Pandas built-in datetime() method. Since there are different date formats, infer_datetime_format = True
          wiki_movies_df['release_date'] = pd.to_datetime(release_date.str.extract(f'({date_form_one}|{date_form_two}|{date_form_three}|{date_form_four})')[0], \   
               infer_datetime_format=True)    

          # Drop Release Date from DataFrame because now there is clean release_date column
          wiki_movies_df.drop('Release date', axis=1, inplace=True)    

          #-----------------------------------------------------------------------------------------------
          # Clean the running time column in the wiki_movies_df DataFrame.
   
          # Create a variable that will hold the non-null values from the “Running time” column and convert to string.
          running_time = wiki_movies_df['Running time'].dropna().apply(lambda x: ' '.join(x) if type(x) == list else x)

          # Extract running time
          running_time_extract = running_time.str.extract(r'(\d+)\s*ho?u?r?s?\s*(\d*)|(\d+)\s*m')
    
          # Convert to numeric values.  Coerce errors: empty strings to NaN and change NaN to zeros with fillna(0)
          running_time_extract = running_time_extract.apply(lambda col: pd.to_numeric(col, errors='coerce')).fillna(0)

          # Convert hours to mins row[0]*60 and add to minutes + row [1]
          wiki_movies_df['running_time'] = running_time_extract.apply(lambda row: row[0]*60 + row[1] if row[2] == 0 else row[2], axis=1) 

          # Drop 'Running Time' from DataFrame because there is clean 'running_time'
          wiki_movies_df.drop('Running time', axis=1, inplace=True)    
    
          #-------------------------------------------------------------------------------------------------
     
          # Clean the Kaggle metadata, keeping only rows where 'adult' is 'False' and then drop 'adult' column.  
          kaggle_metadata = kaggle_metadata[kaggle_metadata['adult'] == 'False'].drop('adult',axis='columns')
    
          # Convert 'video'
          kaggle_metadata ['video'] = kaggle_metadata['video'] == 'True'
    
          # Convert 'budget', 'id', and 'popularity' to numeric.  
          # Set errors = 'raise' in case there's any data that cannot be converted to a number.
          kaggle_metadata['budget'] = kaggle_metadata['budget'].astype(int)
          kaggle_metadata['id'] = pd.to_numeric(kaggle_metadata['id'], errors='raise')
          kaggle_metadata['popularity'] = pd.to_numeric(kaggle_metadata['popularity'], errors='raise')

          # Convert 'release_date' to datetime using pandas
          kaggle_metadata['release_date'] = pd.to_datetime(kaggle_metadata['release_date'])
    
          # Convert ratings 'timestamp' to date/time with Pandas to_datetime(), origin 'unix', time unit=seconds 
          # Assign to the 'timestamp' column
          ratings['timestamp'] = pd.to_datetime(ratings['timestamp'], unit='s')   

          #---------------------------------------------------------------------------------------------------
          # Merge the two DataFrames into the movies DataFrame using Inner Join.
          movies_df = pd.merge(wiki_movies_df, kaggle_metadata, on='imdb_id', suffixes=['_wiki','_kaggle'])

          # Drop unnecessary (i.e., redundant) columns from the merged DataFrame.
          movies_df.drop(columns=['title_wiki','release_date_wiki','Language','Production company(s)'], inplace=True)

          # Add in the function to fill in the missing Kaggle data.
          def fill_missing_kaggle_data(df, kaggle_column, wiki_column):
              df[kaggle_column] = df.apply(
                  lambda row: row[wiki_column] if row[kaggle_column] == 0 else row[kaggle_column], axis=1)
              df.drop(columns=wiki_column, inplace=True)

          # Call the function in step above with the DataFrame and columns as the arguments.
          fill_missing_kaggle_data(movies_df, 'runtime', 'running_time')
          fill_missing_kaggle_data(movies_df, 'budget_kaggle', 'budget_wiki')
          fill_missing_kaggle_data(movies_df, 'revenue', 'box_office')

          # Filter the movies DataFrame for specific columns.
          movies_df = movies_df.loc[:, ['imdb_id','id','title_kaggle','original_title','tagline','belongs_to_collection','url','imdb_link',
                             'runtime','budget_kaggle','revenue','release_date_kaggle','popularity','vote_average','vote_count',
                             'genres','original_language','overview','spoken_languages','Country',
                             'production_companies','production_countries','Distributor',
                             'Producer(s)','Director','Starring','Cinematography','Editor(s)','Writer(s)','Composer(s)','Based on']]

          # Rename the columns in the movies DataFrame.
          movies_df.rename({'id':'kaggle_id',
                        'title_kaggle':'title',
                        'url':'wikipedia_url',
                        'budget_kaggle':'budget',
                        'release_date_kaggle':'release_date',
                        'Country':'country',
                        'Distributor':'distributor',
                        'Producer(s)':'producers',
                        'Director':'director',
                        'Starring':'starring',
                        'Cinematography':'cinematography',
                        'Editor(s)':'editors',
                        'Writer(s)':'writers',
                        'Composer(s)':'composers',
                        'Based on':'based_on'
                       }, axis='columns', inplace=True)

          # Transform and merge the ratings DataFrame, Groupby "movieID" and "rating" columns and take the count.
          # Rename "userID" column to "count".  Pivot data so that movieID is the index and columns will be rating values 
          # and rows will be the counts for each rating value
          rating_counts = ratings.groupby(['movieId','rating'], as_index=False).count() \
                         .rename({'userId':'count'}, axis=1) \
                         .pivot(index='movieId',columns='rating', values='count')   

          # Rename the columns to be easier to understand.
          rating_counts.columns = ['rating_' + str(col) for col in rating_counts.columns]    

          # Left merge movies_df (left) and rating_counts (right) because we want all of movies_df with or without ratings.
          movies_with_ratings_df = pd.merge(movies_df, rating_counts, left_on='kaggle_id', right_index=True, how='left')

          # Fill in missing values with zeros
          movies_with_ratings_df[rating_counts.columns] = movies_with_ratings_df[rating_counts.columns].fillna(0)    

          # Create the connection to the PostgreSQL database "movie_data" created in pgAdmin.
          # Requires 'from sqlalchemy import create_engine' and 'import config.py' (added to line 1 of code)

          db_string = f"postgres://postgres:{db_password}@127.0.0.1:5432/movie_data"
    
          # Create Database Engine
          engine = create_engine(db_string)
    
          # Save movies_df DataFrame to a SQL table using to_sql() method.
          movies_df.to_sql(name='movies', con=engine, if_exists='replace')
    
          # Drop the ratings table in pgAdmin: 
          # https://www.tutorialspoint.com/python_data_access/python_postgresql_drop_table.htm
          # a) import psycopg2 is added to line 1 of code
          # b) create a connection
          conn = psycopg2.connect(database="movie_data", user='postgres', password=db_password, host='127.0.0.1', port= '5432')
    
          # c) Set auto commit (default false)
          conn.autocommit = True
    
          # d) Create a cursor object 
          cursor = conn.cursor()
       
          # e) Drop 'ratings' table
          cursor.execute("DROP TABLE ratings")
          print ("Table dropped....")
    
          # f) Execute the drop table command
          conn.commit()
    
          # g) Free the resources
          cursor.close()
          conn.close()
       
          # Print out the elapsed time to import each row
          # a) create a variable for the number of rows imported
          rows_imported = 0
    
          # b) Get the start_time from time.time()
          start_time = time.time()

          for data in pd.read_csv(f'{file_dir}/ratings.csv', chunksize=1000000):

              # print out the range of rows that are being imported
              print(f'importing rows {rows_imported} to {rows_imported + len(data)}...', end='')

              data.to_sql(name='ratings', con=engine, if_exists='append')

              # increment the number of rows imported by the size of 'data'
              rows_imported += len(data)

              # print that the rows have finished importing
              # add elapsed time to final print out
              print(f'Done. {time.time() - start_time} total seconds elapsed')
 
* Code to create a path to the file directory and variables for the three files:

      file_dir = 'C://Users/kayse/OneDrive/Documents/GitHub/Movies-ETL/Data'  
          
      # The Wikipedia data            
      wiki_file = f'{file_dir}/wikipedia-movies.json'  
          
      # The Kaggle metadata            
      kaggle_file = f'{file_dir}/movies_metadata.csv' 
          
      # The MovieLens rating data.
      ratings_file = f'{file_dir}/ratings.csv'

* Code to call the function to Extract, Transform and Load:

      wiki_movies_df = wiki_file
      movies_with_ratings_df = kaggle_file
      movies_df = ratings_file

      extract_transform_load(wiki_movies_df, movies_with_ratings_df, movies_df)

* Elapsed time to load the clean ratings to the database:
<p align="center">
  <img src="Resources/elapsed_time.png" width="900">
</p>


* postgreSQL Query to count the number of rows imported:
  * `SELECT COUNT (*) FROM  movies`
<p align="center">
  <img src="Resources/movies_query.png" width="300">
</p>

  * `SELECT COUNT (*) FROM  ratings`
<p align="center">
  <img src="Resources/ratings_query.png" width="300">
</p>



[Back to the Table of Contents](https://github.com/rkaysen63/Movies-ETL/blob/master/README.md#table-of-contents)
