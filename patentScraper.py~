import urllib2
import simplejson as json
import requests
from bs4 import BeautifulSoup
import re
import csv
import time
import random
import os.path
import json
import argparse
import io
from pymongo import MongoClient

def parse_arguments():
    parser = argparse.ArgumentParser(conflict_handler="resolve",description="Script to get patent data.")
    parser.add_argument("KEYWORDS_DONE", help="The file which contains the keywords done.")
    parser.add_argument("-not_done","--KEYWORDS_NOT_DONE", help="The file which contains the not done keywords.")
    parser.add_argument("-csv","--CSV_FILE", help="The CSV FILE which contains all the keywords.")
    
    return (parser.parse_args())

#This function gets and stores in a list all the urls given back by the API for a keyword and returns the list.
def get_all_urls(Patent_URLS, next_resultno, query):
    
    try:
        url = 'https://ajax.googleapis.com/ajax/services/search/patent?' + 'v=1.0&q='+ query +'&userip=YOURUSERIP' + "&" + next_resultno
        request = urllib2.Request(url, None, headers = {'User-Agent' : "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Ubuntu Chromium/40.0.2214.111 Chrome/40.0.2214.111 Safari/537.36"})
        response = urllib2.urlopen(request)
        results = json.load(response)
    except Exception as e:
        log_file.write("Error reading the urls for query = " + str(query) + "\n")
        log_file.write(str(type(e)) + " Message:" + str(e.message) + " Args:" + str(e.args) + "\n\n")

    
    #parse the JSON file to get the patent URLS.
    try:
        Patent_URLS.append(re.sub(r"about",results['responseData']['results'][0]['patentNumber'],results['responseData']['results'][0]['unescapedUrl']))
    except:
        pass
    try:
        Patent_URLS.append(re.sub(r"about",results['responseData']['results'][1]['patentNumber'],results['responseData']['results'][1]['unescapedUrl']))
    except:
        pass
    try:
        Patent_URLS.append(re.sub(r"about",results['responseData']['results'][2]['patentNumber'],results['responseData']['results'][2]['unescapedUrl']))
    except:
        pass
    try:
        Patent_URLS.append(re.sub(r"about",results['responseData']['results'][3]['patentNumber'],results['responseData']['results'][3]['unescapedUrl']))
    except:
        pass

    #print (Patent_URLS)
    return (Patent_URLS)

def create_query(keyword, log_file):
    Patent_URLS = []
    query = urllib2.quote(keyword, '')
    try:
        url = 'https://ajax.googleapis.com/ajax/services/search/patent?' + 'v=1.0&q='+ query +'&userip=YOURUSERIP'
        request = urllib2.Request(url, None, headers = {'User-Agent' : "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Ubuntu Chromium/40.0.2214.111 Chrome/40.0.2214.111 Safari/537.36"})
        response = urllib2.urlopen(request)
        results = json.load(response)
    except Exception as e:
        log_file.write("Error fetching the keyword = " + str(keyword) + "\n")
        log_file.write(str(type(e)) + " Message:" + str(e.message) + " Args:" + str(e.args) + "\n\n")


    #uses the Google API again to get all the results, by looping over the start parameter. From the results, urls to the patent pages are extracted.
    try:
        page_startlabels = results['responseData']['cursor']['pages']
        for each_start in page_startlabels:
            next_resultno = "start=" + each_start['start'].encode('utf-8').strip()
            Patent_URLS = get_all_urls(Patent_URLS, next_resultno, query)
    except Exception as e:
        log_file.write("Error reading the results for keyword = " + str(keyword) + "\n")
        log_file.write(str(type(e)) + " Message:" + str(e.message) + " Args:" + str(e.args) + "\n\n")

    return (Patent_URLS)

#This function takes in each_patent's url, scrapes the page and returns the data collected.
def scrape_urls(each_patent, log_file, keyword):
    Data = {}
    try:
        page = requests.get(each_patent)
        page = BeautifulSoup(page.content)
    except Exception as e:
        log_file.write("Error scraping for url = " + str(each_patent) + "\n")
        log_file.write(str(type(e)) + " Message:" + str(e.message) + " Args:" + str(e.args) + "\n\n")
        return Data

    Data['URL'] = each_patent
    Data['Keyword'] = keyword
    try:
        Data['Publication Number'] = page.find(class_="patent-number").get_text().encode('utf-8').strip().replace(" ","", 1)
    except:
        Data['Publication Number'] = "N.A."

    try:
        Data['Patent Title'] = page.find(class_="patent-title").get_text().encode('utf-8').strip().replace("\n"," ")
    except:
        Data['Patent Title'] = "N.A."

    try:
        Data['Abstract'] = page.find(class_="abstract").get_text().encode('utf-8').strip().replace("\n"," ")
    except:
        Data['Abstract'] = "N.A."

    try:
        Data['Description'] = page.find(class_="patent-section patent-description-section").get_text().encode('utf-8').strip().replace("\n"," ")
        Data['Description'] = re.sub("\s+", " ", Data['Description']).strip()
    except:
        Data['Description'] = "N.A."

    try:
        Data['Claims'] = page.find(class_="patent-section patent-claims-section").get_text().encode('utf-8').strip().replace("\n"," ")
        Data['Claims'] = re.sub("\s+", " ", Data['Claims']).strip()
    except:
        Data['Claims'] = "N.A."

    try:
        Data['Thumbnail Image'] = []
        for image in page.find_all(class_="patent-thumbnail-image"):
            Data['Thumbnail Image'].append(image.get('src'))
    except:
        Data['Thumbnail Image'] = []

    try:
        list_of_citations = page.find("a",id="backward-citations").parent.find_all(class_="patent-data-table-td citation-patent")
        Data['CitedPatents'] = []
        for each_citation in list_of_citations:
            each_citation = each_citation.get_text().encode('utf-8').strip()
            Data['CitedPatents'].append(each_citation.replace("*",""))
    except:
        Data['CitedPatents'] = []

    try:
        list_of_referencedby = page.find("a",id="forward-citations").parent.find_all(class_="patent-data-table-td citation-patent")
        Data['ReferencedBy'] = []
        for each_citation in list_of_referencedby:
            each_citation = each_citation.get_text().encode('utf-8').strip()
            Data['ReferencedBy'].append(each_citation.replace("*",""))
    except:
        Data['ReferencedBy'] = []

    try:
        Publication_Date = page.find(text="Publication date")
        Data['Publication Date'] = Publication_Date.find_parent("tr").find(class_="single-patent-bibdata").get_text().encode('utf-8').strip()
    except:
        Data['Publication Date'] = "N.A."

    try:
        Inventors = page.find(text="Inventors")
        list_of_inventors = Inventors.find_parent("tr").find_all(class_="patent-bibdata-value")
        Data['Inventors'] = []
        for each_inventor in list_of_inventors:
            each_inventor = each_inventor.get_text().encode('utf-8').strip()
            Data['Inventors'].append(each_inventor.replace(",",""))
    except:
        Data['Inventors'] = "N.A."

    try:
        Assignee = page.find(text="Original Assignee")
        list_of_assignees = Assignee.find_parent("tr").find_all(class_="patent-bibdata-value")
        Data['Assignee'] = []
        for each_assignee in list_of_assignees:
            each_assignee = each_assignee.get_text().replace(",","").encode('utf-8').strip()
            Data['Assignee'].append(each_assignee.replace(",",""))
    except:
        Data['Assignee'] = "N.A."

    try:
        Applicant = page.find(text="Applicant")
        list_of_applicants = Applicant.find_parent("tr").find_all(class_="patent-bibdata-value")
        Data['Applicant'] = []
        for each_applicant in list_of_applicants:
            each_applicant = each_applicant.get_text().replace(",","").encode('utf-8').strip()
            Data['Applicant'].append(each_applicant.replace(",",""))
    except:
        Data['Assignee'] = "N.A."

    return (Data)

def file_reader(file,keywords_file):
    #This function takes in the file which contains keywords, creates a dictionary of keywords and returns the dicitonary.
    keywords_tempdict = {}
    keywords_dict = {}
    keywords_done = []
    if os.path.isfile(keywords_file):
        keywords_done = json.load(open(keywords_file, 'r'), encoding="utf-8")
   
    with open(file,'r') as readfile:
        reader = csv.DictReader(readfile)
        for row in reader:
            if row['V1'] in keywords_tempdict:
                keywords_tempdict[row['V1']] += 1
            else:
                keywords_tempdict[row['V1']] = 0


    for key in keywords_tempdict:
        if key.decode('utf-8') in keywords_done:
            print "Skipping " + key
        else:
            if key in keywords_dict:
                keywords_dict[key] += 1
            else:
                keywords_dict[key] = 1

    print(len(keywords_dict))
    return (keywords_dict)

def keywords_not_done_reader(keywords_not_done, keywords_donefile):
    keywords_dict = {}
    keywords_tempdict = {}
    keywords_done = []
    
    if os.path.isfile(keywords_donefile):
        keywords_done = json.load(open(keywords_donefile, 'r'), encoding="utf-8")

    readfile = open(keywords_not_done,'r')
    
    for line in readfile.readlines():
        if line in keywords_tempdict:
            keywords_tempdict[line.replace("\n","")] += 1
        else:
            keywords_tempdict[line.replace("\n","")] = 1

    for key in keywords_tempdict:
        if key.decode('utf-8') in keywords_done:
            print "Skipping " + key
        else:
            if key in keywords_dict:
                keywords_dict[key] += 1
            else:
                keywords_dict[key] = 1

    print(len(keywords_dict))
    return (keywords_dict)

def start_operation(start_time, keywords_donefile, csv_file, log_file, keywords_not_done):
    client = MongoClient()
    db = client.patent_database

    keywords_dict = file_reader(csv_file,keywords_donefile)

    keywords_done = []
    if os.path.isfile(keywords_donefile):
        keywords_done = json.load(open(keywords_donefile, 'r'), encoding="utf-8")
    
    for keyword in keywords_dict:
        print (keyword)
        keyword_original = keyword
        keyword = keyword.replace(".","")
        JSON_Data = {}
        JSON_Data[keyword] = []
        Patent_URLS = create_query(keyword, log_file)
        #Data = {}
        for each_patent in Patent_URLS:
            Data = scrape_urls(each_patent, log_file, keyword)
            #Data["Keyword"] = keyword
            JSON_Data[keyword].append(Data)
        
        keywords_done.append(keyword_original)
        end_time = "Keyword: " + keyword + " done. Time: " + str((time.time()-start_time)/60) + " minutes"

        # with open("jsonfile.json", "w") as writefile:
        #     json.dump(JSON_Data, writefile, indent = 4)
        # print ('JSON FILE WRITTEN!')

        db.polymers07042011.save(JSON_Data)
        print ('WRITTEN TO MONGO DB!')

        with open(keywords_donefile, "w") as writefile:
            json.dump(keywords_done, writefile)
        print ('KEYWORDS DONE WRITTEN!')

        print end_time
        delay = 25 + random.random()*10
        some_randomnum = random.randint(1,10)
        delay += random.uniform(0,some_randomnum)
        time_output = "\ndelaying for " + str(delay) + " seconds"
        print time_output
        time.sleep(delay)

def main():
    start_time = time.time()
    args = parse_arguments()
    log_file = open("patent_logfile.txt", "a")

    #print args.KEYWORDS_DONE
    start_operation(start_time, args.KEYWORDS_DONE, args.CSV_FILE, log_file, args.KEYWORDS_NOT_DONE)
    log_file.close()

if __name__ == "__main__":
    main()
