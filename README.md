# hello-world 
**ggmu**
vvgffd


import requests
from bs4 import BeautifulSoup
import json
import csv
import tkinter as tk
from tkinter import ttk, filedialog

class MovieScraperApp:
    def __init__(self, root):
        self.root = root
        self.root.title("IMDb Movie Scraper")
        
        self.sort_options = [
            "num_votes,desc",
            "num_votes,asc",
            "alpha,asc",
            "alpha,desc",
            "release_date,asc",
            "release_date,desc",
            "runtime,asc",
            "runtime,desc"
        ]
        self.selected_sort_option = tk.StringVar()
        
        # Create labels and entry fields for parameters
        self.sort_label = ttk.Label(root, text="Sort Option:")
        self.sort_combobox = ttk.Combobox(root, textvariable=self.selected_sort_option, values=self.sort_options)
        self.sort_combobox.set(self.sort_options[0])  # Set the default value

        # Create a Label for sorting direction indicators
        self.sort_direction_label = ttk.Label(root, text="▼", font=("Arial", 10, "bold"))
        self.sort_direction_label.bind("<Button-1>", self.toggle_sort_direction)  # Bind click event
        
        self.start_label = ttk.Label(root, text="Start Position:")
        self.start_entry = ttk.Entry(root)
        
        self.title_type_label = ttk.Label(root, text="Title Type:")
        self.title_type_entry = ttk.Entry(root)
        
        self.year_range_label = ttk.Label(root, text="Year Range:")
        self.year_range_entry = ttk.Entry(root)
        
        # Set default values
        self.start_entry.insert(0, "1")
        self.title_type_entry.insert(0, "feature")
        self.year_range_entry.insert(0, "1950,2012")
        
        # Create buttons for fetching and exporting data
        self.fetch_button = ttk.Button(root, text="Fetch Data", command=self.fetch_data)
        self.export_json_button = ttk.Button(root, text="Export JSON", command=self.export_to_json)
        self.export_csv_button = ttk.Button(root, text="Export CSV", command=self.export_to_csv)
        
        # Create a Treeview widget to display the table
        self.tree = ttk.Treeview(root, columns=("Title", "Year", "Certificate", "Runtime", "Genre", "Rating", "Director", "Actors"), show="headings")

        # Define column headings
        column_headings = ["Title", "Year", "Certificate", "Runtime", "Genre", "Rating", "Director", "Actors"]
        for col in column_headings:
            self.tree.heading(col, text=col)
            if col in ("Title", "Actors"):
                self.tree.column(col, width=300)  # Adjust the width of the "Title" and "Actors" columns
            else:
                self.tree.column(col, width=100)
        
        # Create a vertical scrollbar
        self.scrollbar = ttk.Scrollbar(root, orient="vertical", command=self.tree.yview)
        self.tree.configure(yscrollcommand=self.scrollbar.set)
        
        # Layout widgets using grid
        self.sort_label.grid(row=0, column=0)
        self.sort_combobox.grid(row=0, column=1)
        self.sort_direction_label.grid(row=0, column=2)
        self.start_label.grid(row=1, column=0)
        self.start_entry.grid(row=1, column=1)
        self.title_type_label.grid(row=2, column=0)
        self.title_type_entry.grid(row=2, column=1)
        self.year_range_label.grid(row=3, column=0)
        self.year_range_entry.grid(row=3, column=1)
        self.fetch_button.grid(row=4, column=0)
        self.export_json_button.grid(row=4, column=1)
        self.export_csv_button.grid(row=4, column=2)
        self.tree.grid(row=5, columnspan=3, sticky="nsew")  # Use sticky to make the Treeview expand to fill the available space
        self.scrollbar.grid(row=5, column=3, sticky="ns")  # Position the scrollbar to the right of the Treeview

        # Allow resizing of columns by dragging
        for col in column_headings:
            self.tree.heading(col, command=lambda c=col: self.sort_column(self.tree, c, False))

    def toggle_sort_direction(self, event):
        current_direction = self.sort_direction_label["text"]
        new_direction = "▲" if current_direction == "▼" else "▼"
        self.sort_direction_label["text"] = new_direction

    def sort_column(self, tree, col, reverse):
        data = [(tree.set(child, col), child) for child in tree.get_children('')]
        data.sort(reverse=reverse)
        for index, (val, child) in enumerate(data):
            tree.move(child, '', index)
        tree.heading(col, command=lambda: self.sort_column(tree, col, not reverse))

    def fetch_data(self):
        sort_option = self.selected_sort_option.get()
        start_position = self.start_entry.get()
        title_type = self.title_type_entry.get()
        year_range = self.year_range_entry.get()
        
        url = generate_imdb_url(sort_option, start_position, title_type, year_range)
        movies_data = scrape_movie_data(url)
        
        # Clear existing data in the tree
        for item in self.tree.get_children():
            self.tree.delete(item)
        
        # Insert new data into the tree
        for movie in movies_data:
            self.tree.insert("", "end", values=list(movie.values()))

    def export_to_json(self):
        data = []
        for item in self.tree.get_children():
            data.append(self.tree.item(item)["values"])

        filename = filedialog.asksaveasfilename(defaultextension=".json", filetypes=[("JSON files", "*.json")])
        if filename:
            json_data = []
            for values in data:
                actors = values[7].split(", ")  # Convert the comma-separated string to a list
                movie_dict = {
                    "Title": values[0],
                    "Year": values[1],
                    "Certificate": values[2],
                    "Runtime": values[3],
                    "Genre": values[4],
                    "Rating": values[5],
                    "Director": values[6],
                    "Actors": actors
                }
                json_data.append(movie_dict)

            with open(filename, "w", encoding="utf-8") as jsonfile:
                json.dump(json_data, jsonfile, ensure_ascii=False, indent=4)

            print(f"Data exported to {filename}")

    def export_to_csv(self):
        data = []
        for item in self.tree.get_children():
            data.append(self.tree.item(item)["values"])

        filename = filedialog.asksaveasfilename(defaultextension=".csv", filetypes=[("CSV files", "*.csv")])
        if filename:
            with open(filename, "w", newline="", encoding="utf-8") as csvfile:
                csv_writer = csv.writer(csvfile)
                csv_writer.writerow(["Title", "Year", "Certificate", "Runtime", "Genre", "Rating", "Director", "Actors"])
                
                for values in data:
                    title = values[0]
                    year = values[1]
                    certificate = values[2]
                    runtime = values[3]
                    genre = values[4]
                    rating = values[5]
                    director = values[6]
                    actors = values[7]

                    csv_writer.writerow([title, year, certificate, runtime, genre, rating, director, actors])

            print(f"Data exported to {filename}")

def generate_imdb_url(sort_option, start_position, title_type, year_range):
    url = f"http://www.imdb.com/search/title?sort={sort_option}&start={start_position}&title_type={title_type}&year={year_range}"
    return url

def extract_actors(stars_elements):
    actors = []
    for element in stars_elements:
        stars = element.find_all("a")
        actors = [star.text for star in stars if not star.text.strip().isdigit() and star.text.strip()]
        if actors:
            break
    return actors

def extract_movie_data(item):
    title = item.find("a").text
    year = item.find("span", class_="lister-item-year").text.strip("()")
    certificate = item.find("span", class_="certificate").text
    runtime = item.find("span", class_="runtime").text
    genre = [g.strip() for g in item.find("span", class_="genre").text.split(",")]
    rating = item.find("strong").text
    director = item.find("p", class_="").find("a").text
    stars_elements = item.find_all("p", class_="")
    actors = extract_actors(stars_elements)
    
    movie_data = {
        "Title": title,
        "Year": year,
        "Certificate": certificate,
        "Runtime": runtime,
        "Genre": genre,
        "Rating": rating,
        "Director": director,
        "Actors": actors
    }
    
    return movie_data

def scrape_movie_data(url):
    response = requests.get(url)
    html_content = response.content
    soup = BeautifulSoup(html_content, "html.parser")

    movies = []
    for item in soup.find_all("div", class_="lister-item-content"):
        movie_data = extract_movie_data(item)
        movies.append(movie_data)
    
    return movies

def save_to_json(data, filename):
    with open(filename, "w", encoding="utf-8") as jsonfile:
        json.dump(data, jsonfile, ensure_ascii=False, indent=4)

def save_to_csv(data, filename):
    with open(filename, "w", newline="", encoding="utf-8") as csvfile:
        fieldnames = data[0].keys()
        writer = csv.DictWriter(csvfile, fieldnames=fieldnames)
        writer.writeheader()
        writer.writerows(data)

def main():
    root = tk.Tk()
    app = MovieScraperApp(root)
    root.mainloop()

if __name__ == "__main__":
    main()

