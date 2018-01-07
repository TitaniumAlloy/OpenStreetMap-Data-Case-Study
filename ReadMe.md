
<h6><font size = 5> OpenStreetMap Data Case Study</font></h6>

Author: Tanbir <br>
Completion Date: November 2017
<br>
<br>
Version: 1.0<br>
Future Updates: N/A until current the Dataset for the map is updated on the site.

<h6><font size = 4> Map Area </font></h6>

Honolulu, Hawaii, United States
<br>Link for map data <a href="https://mapzen.com/data/metro-extracts/">here</a>.

<img src="https://health.hawaii.gov/camhd/files/2013/07/Oahu-FGC-Location-Map.jpg" width="1000" height="500" />

The reason why I chose Honolulu, Hawaii is because Hawaii, the state itself, tends to be a popular spot for vacation for many people in America. What better way for me to learn about the place than looking at the data when there isn't a chance that I will visit there? 

<h6><font size = 4> Problems in the map </font></h6>

After downloading the osm  and running sample_area.py provided, I noticed the file was small and not large enough to detect what problems existed unless we took the full file itself when running audit_count.py. In addition the tags.py presented that there is no problematic character in the file. So when investigating the file I found the following problems:

1. Inconsistent values for addr:street(Street Name). Example: highway, Blvd. etc.
2. Incorrect values addr:street. Example: PI, Penascola etc.
3. Abbreviated Name
4. Inconsistent Phone Number format.
5. Incorrect values for Phone Number.
6. Incorrect postal code.
7. Inconsistent postal code format.

Note that the sample version of the codes are given and outputs are editted for readability purposes. Everything presented here has been written in a notebook.

<h4><font size = 4> Inconsistent, incorrect and abbreviated street name </font></h4>

When running audit_count.py, we can see the several names and dirty data. This is human input so it's expected for these to appear.


```python
#Counts the type of attribute for street value
#The arrow marks appeared after copy and pasting it from the original file.
def audit_street_type(street_types, street_name):
	m = street_type_re.search(street_name)
	if m:
		street_type = m.group()
		street_types[street_type] += 1
		
def print_sorted_dict(d):
	keys = d.keys()
	keys = sorted(keys, key=lambda s: s.lower())
	for k in keys:
		v = d[k]
		print "%s: %d" % (k, v)

def is_street_name(elem):
	return (elem.tag == "tag") and (elem.attrib['k'] == "addr:street")
	
def audit():
	for event, elem in ET.iterparse(osm_file):
        #If the key is addr:street it will audit and count how many times the value occured
		if is_street_name(elem):
			audit_street_type(street_types, elem.attrib['v'])
    #Prints out the set         
	print_sorted_dict(street_types)
```

Executing the code in Jupyter.


```python
exec(open("audit_count.py").read())
```

`83: 1
Ave: 19
Avenue: 273
Blvd: 19
Blvd.: 1
Boulevard: 119
Center: 1
Circle: 13
Dr: 1
Dr.: 1
Drive: 96
Highway: 88
highway: 1
Honolulu: 1
Hwy: 3`

This gives a simplified version of all the types to modify. In audit.py however we see the abbreviated name as well.


```python
exec(open("audit.py").read())
```

` 'Blvd.': set(['Ala Moana Blvd.']),
 'Center': set(['Enchanted Lakes Shopping Center']),
 'Dr': set(['Kipapa Dr']),
 'Honolulu': set(['Moanalua, Honolulu']),
 'Hwy': set(['Kamehameha Hwy']),
 'Hwy.': set(['North Nimitz Hwy.']),
 'Kailua,': set(['Kaelepulu Dr, Kailua,']),
 'KingSt': set(['S KingSt']),
 'Moana': set(['Ala Moana']),
 'Momi': set(['Pali Momi']),
 'Napunani': set(['Ala Napunani']),
 'PI': set(['Kapaa Quarry PI']),
 'Penascola': set(['Penascola']),
 'Pensacola': set(['Pensacola']),`

To fix this and unify values we did the following


```python
expected = ["Street", "Avenue", "Boulevard", "Drive", "Court", "Place", "Square", "Lane", "Road", 
            "Trail", "Parkway", "Commons", "Ike", "Highway", "Way", "83", "Circle","Promenade", "Kailua", "King", "Mall",
           "Loop", "king", "Walk", "Terrace"]

# UPDATE THIS VARIABLE
mapping = { "St": "Street",
            "St.": "Street",
            "Dr" : "Drive",
            "Dr." : "Drive",
            "N." : "North",
            "N" : "North",
            "S" : "South",
            "S." : "South",
            "KingSt" : "King Street",
            "Blvd." : "Boulevard",
            "Blvd" : "Boulevard",
            "PRKWY": "Parkway",
            "Pkwy": "Parkway",
            "Ave": "Avenue",
            "Rd": "Road",
            "highway": "Highway",
            "Hwy.": "Highway",
            "Hwy": "Highway",
            "PI": "Place",
```


```python
#Following code obtain from:
#https://gist.github.com/carlward/54ec1c91b62a5f911c42#file-sample_project-md
#It fixes all the issues with abbreviation, inconsistent, and incorrect values with the mapping and expected
#Note that it does not utilize regular expression
def update(name,mapping):
    words = name.split()
    for w in range(len(words)):
        if words[w] in mapping:
            if words[w-1].lower() not in ['suit', 'ste.', 'ste']:
                words[w] = mapping[words[w]]
    name = " ".join(words)
    return name
```

<h4><font size = 4> Inconsistent and incorrect phone numbers </font></h4>

There are phone numbers that contains strings of alphabets and symbols. In order to standardize this I decide to audit the phone numbers in international format without any symbols. At first I utilized the phonenumbers module which worked really well. Eventually it stopped working for some unknown reason but then again this is a community based module and has lots of dependency so I expected it. I had to write another function utilizing regular expression to get the task done. Examples of the phone number in the original below:
<ul>
  <li>HonoluluCCCampusSecurity:(808)284-1270(cell),(808)271-4836(cell)</li>
  <li>+13234236076</li>
  <li>808.394.8770</li>
  <li>6373000</li>
</ul> 

The code does solve the issues with the phone numbers and unifies it but I decided to manually remove numbers that are empty, over 11 characters and below 11 characters in post audit cleaning in the Database since there aren't many. The numbers can be a mistake so it's best to remove them and unify it. 


```python
#The function below was the original one utilized but stopped working due to unknown reason
#It's an excellent module to use to unify phone numbers in any format.
#https://github.com/daviddrysdale/python-phonenumbers for reference
#def international_phone_format(phone_num):
    #for match in phonenumbers.PhoneNumberMatcher(phone_num, "US"):
       # print phonenumbers.format_number(match.number, phonenumbers.PhoneNumberFormat.E164)
        
#The Function below will remove any special characters from the phone numbers
#In addition adding the number one to make it international format without any special characters in between
def international_phone(phone_num):
    num= phonenum.match(phone_num)
    if num is None:
        #Removes letters from the phone numbers
        phone_num = re.sub("[A-z]", "", phone_num)
        #Removes special characters such as paranthesis
        phone_num = re.sub("[\W]", "", phone_num)
        if "1" not in phone_num[0]:
            phone_num = "1" + phone_num
        # Ignore tag if no area code and local number 
        elif sum(c.isdigit() for c in phone_num) < 10:
            return None
    print phone_num    
    return phone_num                 
        
```


```python
exec(open("phonenumber.py").read())
```

`18085362236
18089237024
18084322000
18087331540
18089266162
18087370177
18089233877
18086742273
18089557470
18882360799
18086377776
18085361330
18085361330
16373000
18083435501`

As I was doing my final step on standardizing the phone numbers a strange occurence happened when the audit was done and moved to the database. Some numbers bigger then 11 remained with white spaces and + sign. However when auditing before transfering in Jupyter those numbers were not there. So I decided to keep the numbers higher then 11 in length to be on the safe side.

`sqlite> SELECT value FROM nodes_tagsC where key='phone' and LENGTH(value)>11;`

`180828412708082714836
+1 808 691 1000
+1 808 922 3861
+1 808 983 6000`

`sqlite> SELECT value FROM nodes_tagsC where key='phone' and LENGTH(value)<11;`

`16373000`

`sqlite> DELETE FROM nodes_tagsC where key='phone' and LENGTH(value)<11;`

`sqlite> DELETE FROM ways_tagsC where key='phone' and LENGTH(value)<11;`

<h4><font size = 4> Incorrect postal code and inconsistent postal code format </font></h4>

Several post code were incorrect and formats were inconsistent:
<ul>
  <li>HI96819</li>
  <li>96797-5640</li>
</ul> 

Utilized regular expression to fix the issue. Any empty zipcode or below 4 length removed in post process.


```python
def clean_postalcode(zipcode):
    #removes special characters from the zip code if any
    zipcode = re.sub("[A-z]", "", zipcode)
    if len(zipcode) > 5:
        zipcode = zipcode[:5]
    
    print zipcode    
    return zipcode       
```


```python
exec(open("audit_postal_code.py").read())
```

`96789
96789
96826
96817
96815
96822
96819
96734
96816
96815
96815`

After cleaning and migrating the postal code, looked for both length above and below of 5. Found one below the standard and removed it that standardized the postal code value.

`sqlite> SELECT value FROM nodes_tagsC where key='postcode' and LENGTH(value)<5;`

`9`

`sqlite> DELETE FROM nodes_tagsC where key='postcode' and value='9';`

<h4><font size = 4> Overview of the data </font></h4>

This section contains basic statistic about the dataset and the SQL queries used to gather them.

<b> File sizes </b>

`Honolulu_Hawaii.osm  69.5 MB
Honolulu_Hawaii.db   46.1 MB
nodesC.csv           27.7 MB
nodes_tagsC.csv      720 KB
waysC.csv            2.04 MB
ways_nodesC.cv       9.59 MB
ways_tagsC.csv       3.96 MB`

<b> Number of ways </b>

`sqlite> SELECT COUNT(*) FROM waysC;`

`35508`

<b>Number of nodes</b>

`sqlite> SELECT COUNT(*) FROM nodesC;`

`340614`

<b> Number of unique users </b>

`sqlite> SELECT COUNT(DISTINCT(uid))          
FROM (SELECT uid FROM nodesC UNION ALL SELECT uid FROM waysC);`

`607`

<b> Number of total user post based on users appearing in the data </b>

`sqlite> SELECT COUNT(uid)
   ...> FROM (SELECT uid FROM nodesC UNION ALL SELECT uid FROM waysC);`

`376122`

<b> Top 15 user post </b>

`sqlite> SELECT user, COUNT(*) as num
   ...> FROM (SELECT user FROM nodesC UNION ALL SELECT user FROM waysC)
   ...> GROUP BY user
   ...> ORDER BY num DESC
   ...> LIMIT 15;`

`Tom_Holland|90288
cbbaze|32046
OklaNHD|29320
dufekin|24114
fscio|17835
julesreid|15354
ikiya|12237
kr4z33|11638
abishek_magna|11564
Chris Lawrence|9111
pdunn|8310
aaront|7097
woodpeck_fixbot|6417
bdiscoe|4726
Mele Sax-Barnett|4594`


```python
90288.00 / 376122.00
```




    0.24004977108491393




```python
32046/376122.00
```




    0.08520107837350648



<h4><font size = 4> Ideas about the dataset</font></h4>

So far based on the overview dataset queries we did, we can say that the contribution is somewhat skewed with the top person having 24% estimated posts while the second place having 9 percent estimated. However this is to be expected because factors are involved such as, is the person living there who submitted it, did the person move that submitted, did anyone know about the site, and etc. With that many people putting data, there is bound to be people with different formats. This is what we have seen. The fact that they're were people posting incorrect road name and formats, I recommend putting a standard format checkers where the data being input by the user will be checked. If the format matches the top contributer or if a standard template format that is compared with the user submitted data and constraints them from inputting wrong format of data based on it, then it would reduce the maybe even remove the inconsistent data. No matter what ideas we input about increasing the contribution for the OpenStreetmap, there is no way we can have a unified and standard data if we don't have at least constraints which are checked before user data is submitted. As we saw earlier, there were data that had string characters where number numbers were suppose to belong.
<br>
<br>
The difficulty with this however is the streets may change over time and the nodes will be effected within that time frame. In addition the place may not be the same so there could be a change in the format. So keeping a standard template for a zone may not be sufficient over time. In addition many places are different in way they have roads, highway, etc. Some places have traffic circles where others have roundabout. In other words there would be lots of templates to be produced and maintained if that idea ever were to be put forth in order to keep user input data. There also maybe a case where someone types a wrong string character name for a street and it will pass the constraint check.

<h4><font size = 4> Additional Data exploration </font></h4>

<img src="http://media.royalcaribbean.com/content/shared_assets/images/ports/hero/HNL_01.jpg">

<b> Religion in the dataset </b>

`sqlite> SELECT nodes_tagsC.value, COUNT(*) as num
FROM nodes_tagsC 
    JOIN (SELECT DISTINCT(id) FROM nodes_tagsC WHERE value='place_of_worship') i
    ON nodes_tagsC.id=i.id
WHERE nodes_tagsC.key='religion'
GROUP BY nodes_tagsC.value
ORDER BY num DESC;		`

`christian|8
buddhist|2
pagan|1`

<b> Bank information </b>

`sqlite> SELECT nodes_tagsC.value, COUNT(*) as num
        FROM nodes_tagsC
            JOIN (SELECT DISTINCT(id) FROM nodes_tagsC WHERE value='bank') i
            ON nodes_tagsC.id=i.id
        WHERE nodes_tagsC.key='name'
        GROUP BY nodes_tagsC.value
        ORDER BY num DESC;`

`First Hawaiian Bank|7
Bank of Hawaii|6
American Savings Bank|5
Bank Of Hawaii|2
Central Pacific Bank|2
Bank of Hawaii Headquarters Main Branch|1
Central Pacific Bank - Waikiki Branch|1
FNL Insurance Company, LTD|1
First Hawaiian Bank Headquarters Branch|1
First Hawaiian Bank-Kapolei Branch|1
Hawaiian Airlines FCU|1
Territorial Savings Bank|1`

The bank information is really interesting. Since I live in New York City, I expected at least one of the major branch of banks from the city to be located there. Seems like they have they're own bank. 

<h4><font size = 4> Conclusion </font></h4>

The dataset is small and messy. There are also missing places that aren't presented in the data so it's clear the data isn't up to date and accurate. The data is not 100% clean but I believe it was sufficiently cleaned for this project. I learned many things about Hawaii even though I never went there in my life. However there is improvements to be made for this data in the future.
