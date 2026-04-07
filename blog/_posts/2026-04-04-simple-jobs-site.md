---
layout: post
title:  "Simple Job Listings App with SerpAPI"
date:   2026-04-01
categories:
    - python
    - serpAPI
toc: true
---
Build a simple job search app using SerpAPI, Postgres, and Python.

[Preview the demo](https://hamcclellan.pythonanywhere.com/)

[Jump to the Git repo](https://github.com/harchermcclellan/job-finder)

![Project screen grab](/assets/projects/Job-finder.png)

## Getting Started

To complete this project, you'll need Python set up, possibly in a virtual environment.

To start, set up a directory called job-finder wherever you like to build projects (for me, that's `/Desktop/dev`, so I would start with `cd /Desktop/dev`):
```
mkdir job-finder
cd job-finder
python3.12 -m venv ./.venv
source .venv/bin/activate
```

Here's what we just did:
`mkdir {name}` creates a new directory within our current location, with whatever name we give it
`cd {path}` changes our current location to whatever path we give it. To start from our user directory, we can use `cd /`. To get to the very top of our directory tree, we can use `cd ~`. To go up a level, we can use `cd .`, and to navigate from where we currently are, we just name the subdirectory within the folder we're in. So we've navigated from our home directory to `Desktop` to the `dev` directory, created `job-finder` within it, and jumped into the `job-finder` directory.

Finally, we're specifying that we want to set up a python virtual environment built in python 3.12, and the configuration for that virtual environment will live in the directory named `.venv` within the  `job-finder` directory.

Finally, we want to activate our virtual environment so we can install some packages and get things running, so we run the activate method

Now that we have our virtual environment set up, we want to install a few tools we'll be using along the way:

```
pip install flask
pip install dotenv
pip install serpapi
pip install psycopg2
```

`pip` is Python's package installer -- most tools that are built with a python API are distributed using pip. It's best not to install packages without any idea what they are:
flask is a framework that allows you to build web apps
serpapi is the tool we're really interested in - it scrapes google results and sets APIs we can use to access those scrapes.
psycopg2 is an adapter for PostgreSQL databases into Python. Basically, it enables us to use Python to run Postgres commands.

We won't use these tools as we're just getting started, but we'll need them soon.

## Getting online

We want to create a basic homepage within a new folder:
```
mkdir templates
touch index.html
touch app.py
```

`touch {name}` creates a new file with the specified name. Our interface will live in `index.html` and our logic and api calls will live in `app.py`

For now, let's put just the basics in our html:

```
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Job Finder</title>
  <link rel="preconnect" href="https://fonts.googleapis.com" />
  <link href="https://fonts.googleapis.com/css2?family=DM+Mono:ital,wght@0,300;0,400;0,500;1,400&family=Syne:wght@400;600;700;800&display=swap" rel="stylesheet" />
 
</head>
<body>
Hello
</body>
```

and in our Python file:
```
if __name__ == "__main__":
    app.run(debug=True)

@app.route("/")
def index():
    return render_template("index.html")

```

This tells our site to run when we run it from the command line, and that when it hits `/`, to build the html from index.html


### Set up API and database access
We're going to use SerpAPI to access google search results, and want to cache those search results to limit our API usage.

- If necessary, create a SerpAPI account. Locate your API key:
![API key in SerpAPI location](/blog/images/FindAPIkey.png)
In the next step, we'll refer to this as `APIKEY`
- Create an account with a database provider. (I used [Supabase](https://supabase.com) because setup is easy and I'm comfortable using Postgres). Within your database provider, find your database connection string. We'll refer to this as `DB_CONN` in the next step

### Set environment variables
Determine whether you have a bashrc file configured yet:

- In your terminal, try `open ~/.bashrc`
  - If you get the error "The file /Users/[your user]/.bashrc does not exist" you'll need to create it first:
    - `touch ~/.bashrc`
    - As in the earlier steps, `touch` creates a file for us
    - Then, open the file. Run `open ~/.bashrc` again
- To the end of the file, add two lines:
  - `export SERP_API_KEY="{APIKEY}"` where `APIKEY` is the key from the above step. So, if your SerpAPI key was "baluga", you'd add `export SERP_API_KEY="baluga"`
  - `export DATABASE_URL="{DB_CONN}"` using `DB_CONN` from the step above
- Save the file and close it.
- Activate the bash profile:
 - `source ~/.bashrc`
- Verify the profile is active:
  - `echo $DATABASE_URL` should return the value of your database URL

## Create the frontend
Can I tell you a secret? I don't love writing frontend code. Like, I don't enjoy it at all. So, humbly, I prompted Claude to do it. To do this successfully, be very specific about what you want, what the use case is, and what technical specifications you'll need. Here are some parts of my prompt:
- Create the frontend for a job search single-page app with the following inputs:
  - Title, location, search radius
- I'll define the API calls to retrieve data myself

My final html file can be found here for reference, but try generating one yourself! Below, I'll discuss some changes I made for my own purposes.

## Set up our app backend
We'll need a `GET` method that renders the webpage, a way to run the app locally, and as defined in the webpage itself, we'll also need a `POST` that retrieves data to be displayed.
### Running the app from the terminal
- At the bottom of your app.py file, add the following:

```
app = Flask(__name__)

if __name__ == "__main__":
    app.run(debug=True)
```

  This tells our terminal that if the name of the program is __main__, run the app.

### Add a `GET` method
- At the bottom of the app.py file, add the following:

```
@app.route("/")
def index():
    return render_template("index.html")
```

This tells the app that when the root is hit (when we access "/" from our URL), to render index.html


### Add a `POST` method

#### Create a placeholder response
- Let's start by setting up sample data for our POST method to return. The following is data that a SerpAPI call returned for a search for "designer" jobs in Boston, MA, and it'll make great basic data for our app to display in demo mode:

```
"jobs_results": [
            {
            "title": "Graphic Designer",
            "company_name": "Cell Signaling Technology",
            "location": "Danvers, MA",
            "via": "Indeed",
            "share_link": "https://www.google.com/search?ibp=htl;jobs&q=designer&htidocid=phJPm6Ds4FE-sgoFAAAAAA%3D%3D&hl=en-US&shndl=37&shmd=H4sIAAAAAAAA_xXNsQrCMBAAUFz7CU63KaKNCC46iYWC4KR7uYYjiZx3IRek_oWfrC5vfc1n1iz6gjkmDx1ZCkIFNnDREYyw-Agq0KsGpvkx1prt4JwZt8Eq1uRbr0-nQqNO7qGj_RksYqHMWGnY7bdTmyWslmdihtsvQE4S4E4-irKGNySBDuVFxdZwPX0BpGvV45MAAAA&shmds=v1_ATWGeePq5XgzE3Hqc42KRV6RDJvhyhd8shB1SEqAQuWFTB5qVg&source=sh/x/job/li/m1/1#fpstate=tldetail&htivrt=jobs&htiq=designer&htidocid=phJPm6Ds4FE-sgoFAAAAAA%3D%3D",
            "thumbnail": "https://serpapi.com/searches/69c6ca4931aafe3e972a6c09/images/7q95LRhnXUo8AuAIEafrXQiw7vuLwWstWp1UxOPYHwo.jpeg",
            "extensions": [
                "3 days ago",
                "81K–120K a year",
                "Full-time",
                "Paid time off",
                "Health insurance"
            ],
            "detected_extensions": {
                "posted_at": "3 days ago",
                "salary": "81K–120K a year",
                "schedule_type": "Full-time",
                "paid_time_off": True,
                "health_insurance": True
            },
            "source_link": "https://www.indeed.com/viewjob?jk=a9b71d0d25e1de09",
            "description": "Who we are…\n\nCell Signaling Technology (CST) is a different kind of life sciences company, one founded, owned, and run by active research scientists, with the highest standards of product and service quality, technological innovation, and scientific rigor for over 20 years. We consistently provide fellow scientists around the globe with best-in-class products and services to fuel their quests for discovery.\n\nHelping researchers find new solutions is our main mission every day, but it's not our only mission. We're also dedicated to helping identify solutions to other problems facing our world. We believe that all businesses must be responsible and work in partnership with local communities, while seeking to minimize their environmental impact. That's why we joined 1% for the Planet as its first life science member, and have committed to achieving net-zero emissions by 2029.\n\nThe role...\n\nThe Graphic Designer is a core member of the design team, reporting directly to the Creative Director. In this role, you will collaborate with a multidisciplinary team of marketing and communication specialists to conceptualize and produce high-impact visual assets across a variety of digital and traditional media formats. This is an on-site position based in Danvers, MA.\n\nYou'll have the opportunity to...\n\nProject Planning & Collaboration (15%)\n• Cross-Functional Partnership: Collaborate with Marketing Managers, Product Management, Scientific Writers, and Regional Managers to define project scope, goals, and deliverables\n• Autonomy & Ownership: Work with minimal supervision to manage the end-to-end design process, from initial conceptualization through final production and implementation\n• Project Management: Simultaneously manage multiple project timelines and coordinate with internal teams and external vendors to ensure timely delivery\n\nGraphic & Visual Design (65%)\n• Interactive Content & Motion Graphics: Design and produce compelling motion graphics, animated elements, and short-form video for use across social media channels, website content, and digital advertising, ensuring assets are optimized for engagement and modern digital standards\n• Proficient Toolset: Utilize a professional suite of tools, including Adobe Creative Cloud (Illustrator, InDesign, Photoshop, Premiere), and web publishing tools (HTML5, PDF)\n• Visual Assets: Create and curate custom iconography, photographic imagery, and illustrations that meet modern digital standards\n• Strategic Optimization: Actively research best practices and measure campaign impact to recommend design changes that improve performance and reduce production effort\n\nCorporate Branding & Identity (5%)\n• Brand Stewardship: Act as a guardian of the corporate style guide, ensuring all materials maintain a consistent, professional \"CST\" look and feel\n• Template Development: Build and maintain standardized Marcom templates to streamline design workflows while ensuring brand alignment\n• Identity Materials: Design core corporate identity assets, including stationery, business cards, logos, and global presentation templates\n\nPackaging & Product Design (5%)\n• Packaging Solutions: Conceptualize and design physical product packaging, including containers, sleeves, labels, and mailing envelopes\n• Strategic Alignment: Partner with Product Management to ensure all packaging designs meet both brand strategy requirements and strict regulatory standards\n\nMaintenance, Implementation & Support (10%).\n• Ongoing Support: Manage the maintenance and implementation of ongoing graphic projects, adapting materials for various formats, channels, and global regions as needed\n• Trend Innovation: Continuously research and implement new graphic design techniques, formats, and trends to keep visual content fresh and competitive\n\nWho you are and what you bring to the team...\n• Bachelor's degree in Graphic Design, Visual Arts, or a related field\n• Minimum of 5 years of related professional graphic design experience, preferably within a corporate or technical/scientific environment.\n• Highly proficient in Adobe Creative Suite (InDesign, Photoshop, Illustrator, Acrobat)\n• Knowledge of Adobe Premiere or comparable video editing software\n• Experience with creating compelling 2D & 3D animations\n• Experience with the four-color printing process, including preparing print files and overseeing press checks\n• Proficiency in Google Slides and Microsoft Office PowerPoint for global sales templates\n• Proven ability to manage the end-to-end design process (concept to finish) with minimal supervision\n• Strong organizational skills to manage multiple projects simultaneously under tight deadlines\n• Excellent communication skills with the ability to work across departments and international regions; confidence in presenting and explaining creative ideas\n• Excellent communication and content development skills\n• Ability to work cooperatively with many personalities and cultures\n• Ability to work independently\n\nIdeally, you have...\n• Experience in creative concept development for print, digital advertising, social media\n• Experience in developing compelling graphics and content for trade show and events environments\n• Experience in video production: shooting, editing, animating\n• Experience with creating compelling 2D & 3D animations\n• Experience with digital asset managers like Bynder\n• Experience creating and managing brand style guides\n• Experience in developing and editing Powerpoint and Google Slides templates and content\n\nPhysical Conditions/Physical Requirements...\n• Work environment is typically quiet for independent desk work and virtual meetings\n• The ability to perform repetitive motions with the wrists, hands, and fingers.\n• The ability to extend hands and arms in any direction to access materials, equipment, or a keyboard\n• The ability to sit for long periods of time (often more than 70% of the workday)\n• Ability to view and read information from a computer monitor, reports, and documents\n• Physically able to safely lift and carry 25 pounds to move, install, unpack PC equipment, office supplies, etc. as required\n\nThis position has a starting base salary range of $81,000 to $120,000 per year, which the company, in good faith, reasonably expects to pay for this role at the time of posting. Actual compensation within this range will be determined based on factors including, but not limited to, relevant skills, qualifications, experience, and internal equity.\n\nWhat we offer...\n\nAt Cell Signaling Technology (CST), we recognize that people will always be our most important asset. Providing a safe, inclusive, and stimulating working environment that understands the importance of diversity, human dignity, and meaningful work is as important as establishing company policies that incorporate excellent health insurance and pay benefits. We recognize that the development of people is the key to their happiness and thus ensure every employee has impactful discussions with their manager and develops actionable performance and professional development plans. Lastly, we are committed to engaging and supporting our employees in committees and philanthropy that benefit their local communities and environment through community investment programs.\n\nBenefits\n• Medical (BCBS) and Dental (Delta Dental) plans paid at 90%\n• Vision Insurance\n• Life Insurance, Short and Long Term Disability\n• Flexible Spending accounts\n• 401(k) Plan with 6% match\n• Tuition Reimbursement\n• Generous PTO package\n• Parental Leave\n• Pet Insurance\n• Employee Assistance Program\n• Onsite Subsidized Cafeteria\n• Free Parking\n\nCell Signaling Technology (CST) is committed to providing equal employment opportunities to all employees and applicants for employment without regard to race, color, religion, sex, sexual orientation, national origin, age, disability, genetic information, status as a veteran or as a member of the military or status in any group protected by applicable federal or state laws.\n\nIt is unlawful in Massachusetts to require or administer a lie detector test as a condition of employment or continued employment. An employer who violates this law shall be subject to criminal penalties and civil liability.\n\nAGENCIES\n\nAll resumes submitted by search firms/employment agencies to any employee at Cell Signaling Technology (CST) via email, the Internet, or in any form and/or method will be deemed the sole property of CST unless CST engaged such search firms/employment agencies for this position and a valid agreement with CST is in place. In the event a candidate who was submitted outside of the CST agency engagement process is hired, no fee or payment of any kind will be paid.\n\nIf you are a California resident, more details on how we process your personal information can be found here.",
            "job_highlights": [
                {
                "title": "Qualifications",
                "items": [
                    "Interactive Content & Motion Graphics: Design and produce compelling motion graphics, animated elements, and short-form video for use across social media channels, website content, and digital advertising, ensuring assets are optimized for engagement and modern digital standards",
                    "Proficient Toolset: Utilize a professional suite of tools, including Adobe Creative Cloud (Illustrator, InDesign, Photoshop, Premiere), and web publishing tools (HTML5, PDF)",
                    "Bachelor's degree in Graphic Design, Visual Arts, or a related field",
                    "Minimum of 5 years of related professional graphic design experience, preferably within a corporate or technical/scientific environment",
                    "Highly proficient in Adobe Creative Suite (InDesign, Photoshop, Illustrator, Acrobat)",
                    "Knowledge of Adobe Premiere or comparable video editing software",
                    "Experience with creating compelling 2D & 3D animations",
                    "Experience with the four-color printing process, including preparing print files and overseeing press checks",
                    "Proficiency in Google Slides and Microsoft Office PowerPoint for global sales templates",
                    "Proven ability to manage the end-to-end design process (concept to finish) with minimal supervision",
                    "Strong organizational skills to manage multiple projects simultaneously under tight deadlines",
                    "Excellent communication skills with the ability to work across departments and international regions; confidence in presenting and explaining creative ideas",
                    "Excellent communication and content development skills",
                    "Ability to work cooperatively with many personalities and cultures",
                    "Ability to work independently",
                    "Experience in creative concept development for print, digital advertising, social media",
                    "Experience in developing compelling graphics and content for trade show and events environments",
                    "Experience in video production: shooting, editing, animating",
                    "Experience with creating compelling 2D & 3D animations",
                    "Experience with digital asset managers like Bynder",
                    "Experience creating and managing brand style guides",
                    "Experience in developing and editing Powerpoint and Google Slides templates and content",
                    "Physical Conditions/Physical Requirements..",
                    "Work environment is typically quiet for independent desk work and virtual meetings",
                    "The ability to perform repetitive motions with the wrists, hands, and fingers",
                    "The ability to extend hands and arms in any direction to access materials, equipment, or a keyboard",
                    "The ability to sit for long periods of time (often more than 70% of the workday)",
                    "Ability to view and read information from a computer monitor, reports, and documents",
                    "Physically able to safely lift and carry 25 pounds to move, install, unpack PC equipment, office supplies, etc. as required"
                ]
                },
                {
                "title": "Benefits",
                "items": [
                    "This position has a starting base salary range of $81,000 to $120,000 per year, which the company, in good faith, reasonably expects to pay for this role at the time of posting",
                    "Medical (BCBS) and Dental (Delta Dental) plans paid at 90%",
                    "Vision Insurance",
                    "Life Insurance, Short and Long Term Disability",
                    "Flexible Spending accounts",
                    "401(k) Plan with 6% match",
                    "Tuition Reimbursement",
                    "Generous PTO package",
                    "Parental Leave",
                    "Pet Insurance",
                    "Employee Assistance Program",
                    "Onsite Subsidized Cafeteria",
                    "Free Parking"
                ]
                },
                {
                "title": "Responsibilities",
                "items": [
                    "The Graphic Designer is a core member of the design team, reporting directly to the Creative Director",
                    "In this role, you will collaborate with a multidisciplinary team of marketing and communication specialists to conceptualize and produce high-impact visual assets across a variety of digital and traditional media formats",
                    "Project Planning & Collaboration (15%)",
                    "Cross-Functional Partnership: Collaborate with Marketing Managers, Product Management, Scientific Writers, and Regional Managers to define project scope, goals, and deliverables",
                    "Autonomy & Ownership: Work with minimal supervision to manage the end-to-end design process, from initial conceptualization through final production and implementation",
                    "Project Management: Simultaneously manage multiple project timelines and coordinate with internal teams and external vendors to ensure timely delivery",
                    "Graphic & Visual Design (65%)",
                    "Visual Assets: Create and curate custom iconography, photographic imagery, and illustrations that meet modern digital standards",
                    "Strategic Optimization: Actively research best practices and measure campaign impact to recommend design changes that improve performance and reduce production effort",
                    "Brand Stewardship: Act as a guardian of the corporate style guide, ensuring all materials maintain a consistent, professional \"CST\" look and feel",
                    "Template Development: Build and maintain standardized Marcom templates to streamline design workflows while ensuring brand alignment",
                    "Identity Materials: Design core corporate identity assets, including stationery, business cards, logos, and global presentation templates",
                    "Packaging & Product Design (5%)",
                    "Packaging Solutions: Conceptualize and design physical product packaging, including containers, sleeves, labels, and mailing envelopes",
                    "Strategic Alignment: Partner with Product Management to ensure all packaging designs meet both brand strategy requirements and strict regulatory standards",
                    "Maintenance, Implementation & Support (10%)",
                    "Ongoing Support: Manage the maintenance and implementation of ongoing graphic projects, adapting materials for various formats, channels, and global regions as needed",
                    "Trend Innovation: Continuously research and implement new graphic design techniques, formats, and trends to keep visual content fresh and competitive"
                ]
                }
            ],
            "apply_options": [
                {
                "title": "Indeed",
                "link": "https://www.indeed.com/viewjob?jk=a9b71d0d25e1de09&utm_campaign=google_jobs_apply&utm_source=google_jobs_apply&utm_medium=organic"
                },
                {
                "title": "Glassdoor",
                "link": "https://www.glassdoor.com/job-listing/graphic-designer-cell-signaling-technology-JV_IC1154555_KO0,16_KE17,42.htm?jl=1010076102515&utm_campaign=google_jobs_apply&utm_source=google_jobs_apply&utm_medium=organic"
                },
                {
                "title": "LinkedIn",
                "link": "https://www.linkedin.com/jobs/view/graphic-designer-at-cell-signaling-technology-cst-4389773865?utm_campaign=google_jobs_apply&utm_source=google_jobs_apply&utm_medium=organic"
                },
                {
                "title": "ZipRecruiter",
                "link": "https://www.ziprecruiter.com/c/Cell-Signaling-Technology/Job/Graphic-Designer/-in-Danvers,MA?jid=e09bbc4f31d38e04&utm_campaign=google_jobs_apply&utm_source=google_jobs_apply&utm_medium=organic"
                },
                {
                "title": "The Design Project",
                "link": "https://designproject.io/jobs/jobs/graphic-designer-at-cell-signaling-technology-im60tr?utm_campaign=google_jobs_apply&utm_source=google_jobs_apply&utm_medium=organic"
                },
                {
                "title": "Teal",
                "link": "https://www.tealhq.com/job/graphic-designer_7ea1ab5c72925f3ac66ac0cbef4de76a9e1ff?utm_campaign=google_jobs_apply&utm_source=google_jobs_apply&utm_medium=organic"
                },
                {
                "title": "Built In",
                "link": "https://builtin.com/job/graphic-designer/8854945?utm_campaign=google_jobs_apply&utm_source=google_jobs_apply&utm_medium=organic"
                },
                {
                "title": "SimplyHired",
                "link": "https://www.simplyhired.com/job/yNtOww_Fl8OfurNDmU9E45ZpJazoVKzgSs4-Ra8DLXSYn1otnkHoWA?utm_campaign=google_jobs_apply&utm_source=google_jobs_apply&utm_medium=organic"
                }
            ],
            "job_id": "eyJqb2JfdGl0bGUiOiJHcmFwaGljIERlc2lnbmVyIiwiY29tcGFueV9uYW1lIjoiQ2VsbCBTaWduYWxpbmcgVGVjaG5vbG9neSIsImFkZHJlc3NfY2l0eSI6IkRhbnZlcnMsIE1BIiwiaHRpZG9jaWQiOiJwaEpQbTZEczRGRS1zZ29GQUFBQUFBPT0iLCJ1dWxlIjoidytDQUlRSUNJaE1ESXhORFFzVFdGemMyRmphSFZ6WlhSMGN5eFZibWwwWldRZ1UzUmhkR1Z6IiwiZ2wiOiJ1cyIsImhsIjoiZW4ifQ=="
            },
            {
            "title": "Designer",
            "company_name": "CRATE & BARREL",
            "location": "Natick, MA",
            "via": "Jobs And Careers At CRATE & BARREL",
            "share_link": "https://www.google.com/search?ibp=htl;jobs&q=designer&htidocid=kicsPssnbAPDw-c_AAAAAA%3D%3D&hl=en-US&shndl=37&shmd=H4sIAAAAAAAA_xXMMQsCIRQA4P1-QgS9qSFKI2ipyeoIohqk_VB5qGU-8Tnc0I-PW77x637d4oIcfcYKG7iRBUZTXQDKcCXyCWfH0Frhg5TMSXhupkUnHH0lZbQ0yjdZnhg4mIolmYbDbr8dRcl-NT9r9ephCSeldX-HmOE5BZ81PNQfaHEah34AAAA&shmds=v1_ATWGeeMm9M8qTQpNg34Q1qL_jts7RuzuSJ99-83VXPd1PGIM7A&source=sh/x/job/li/m1/1#fpstate=tldetail&htivrt=jobs&htiq=designer&htidocid=kicsPssnbAPDw-c_AAAAAA%3D%3D",
            "thumbnail": "https://serpapi.com/searches/69c6ca4931aafe3e972a6c09/images/J3jYS_JGEZWiKz8UnJy-lFhHT-IHwdLoiJfZu7ctm7M.jpeg",
            "extensions": [
                "Part-time",
                "No degree mentioned"
            ],
            "detected_extensions": {
                "schedule_type": "Part-time",
                "qualifications": "No degree mentioned"
            },
            "source_link": "https://jobs.crateandbarrel.com/job/natick/designer/351/90551819568",
            "description": "Crate and Barrel Designers are passionate about helping customers envision possibilities with the latest home design trends. They build meaningful, long-term relationships by using their knowledge to guide customers in furnishing anything from an entire home to a single accent piece. Skilled across a range of design styles—from classic to contemporary—Designers utilize digital tools and technology during in-store and in-home consultations to bring customer visions to life. In this role, you will drive sales and customer engagement by promoting programs, leveraging leads, and maintaining an active presence on the salesfloor. You will conduct customer outreach, develop design packages to brand standards, and ensure timely follow-up. Maintaining operational excellence through impeccable product presentation and careful use of tools and technology is essential, as is collaborating with store and design teams to support business goals. This role offers a creative, rewarding career path for those passionate about home interiors and thriving in a team-oriented, competitive environment.\n\nA day in the life as a Designer...\n• Drive sales and a differentiated experience by providing enriching customer interactions, and providing elevated design offerings in the store, in-home and virtually with customers\n• Create elevated designs for customers using the preferred design tools to create moodboards, 2D & 3D floor plans, product lists and customer presentations\n• Lead design consultations in person (in-store or in-home) or via email, phone and virtual\n• Deliver projects in a timely manner and within determined timelines\n• Possess a clear understanding of the brand aesthetics and merchandising strategy by channel; Store, E-Commerce, Catalog\n• Ensure full understanding and awareness of all product information, including characteristics, care information and staying informed with the competition and industry trends\n• Deliver individual sales, KPI and service goals, productivity standards, and engage customers on the sales floor by demonstrating our selling skills\n• Actively listen to the customer to identify which products will best meet their needs and communicate company loyalty services. (e.g. designer rewards, Design Trade Program, credit card etc)\n• Support and model excellent service by exhibiting a positive attitude and enthusiasm ensuring all customers are provided gracious, quick, and efficient service\n• Support store training and educating on design services, to drive a clear understanding of design services and offerings\n• Develop new and lasting relationships with customers through networking and clienteling\n• Understanding of basic design functions including spatial planning, fabric selection, lighting, interior design styles\n• Excellent, effective, and timely communication skills and the ability to translate the brand vision and the customers wants/needs\n• Strong affinity for technology (2D and 3D tools, Google suite, video conferencing, iPad) and proficient in floor planning\n• Ability to stay up to date on current design trends\n• Ability to be an agent of change and shift quickly as our business evolves\n\nWe'd love to hear from you if you have…\n• Understanding of basic design functions including spatial planning, fabric selection, lighting, interior design styles\n• Excellent, effective, and timely communication skills and the ability to translate the brand vision of CB2 and the customers wants/needs\n• Strong affinity for technology (2D and 3D tools, Google suite, video conferencing) and proficient in floor planning\n• Ability to stay up to date on current design trends\n• Proven track record of building long-lasting relationships with customers\n\nWe'd love to hear from you if you have…\n• 1+ years of relevant experience in Furniture Sales/ Home Decor Design or retail/ customer service experience\n• Experience working one on one with clients and recommending solutions\n• Proficient in Google platforms, virtual communication, design tool experience preferred",
            "job_highlights": [
                {
                "title": "Qualifications",
                "items": [
                    "Excellent, effective, and timely communication skills and the ability to translate the brand vision and the customers wants/needs",
                    "Strong affinity for technology (2D and 3D tools, Google suite, video conferencing, iPad) and proficient in floor planning",
                    "Ability to stay up to date on current design trends",
                    "Ability to be an agent of change and shift quickly as our business evolves",
                    "Understanding of basic design functions including spatial planning, fabric selection, lighting, interior design styles",
                    "Excellent, effective, and timely communication skills and the ability to translate the brand vision of CB2 and the customers wants/needs",
                    "Strong affinity for technology (2D and 3D tools, Google suite, video conferencing) and proficient in floor planning",
                    "Ability to stay up to date on current design trends",
                    "Proven track record of building long-lasting relationships with customers",
                    "1+ years of relevant experience in Furniture Sales/ Home Decor Design or retail/ customer service experience",
                    "Experience working one on one with clients and recommending solutions"
                ]
                },
                {
                "title": "Responsibilities",
                "items": [
                    "They build meaningful, long-term relationships by using their knowledge to guide customers in furnishing anything from an entire home to a single accent piece",
                    "Skilled across a range of design styles—from classic to contemporary—Designers utilize digital tools and technology during in-store and in-home consultations to bring customer visions to life",
                    "In this role, you will drive sales and customer engagement by promoting programs, leveraging leads, and maintaining an active presence on the salesfloor",
                    "You will conduct customer outreach, develop design packages to brand standards, and ensure timely follow-up",
                    "Maintaining operational excellence through impeccable product presentation and careful use of tools and technology is essential, as is collaborating with store and design teams to support business goals",
                    "Drive sales and a differentiated experience by providing enriching customer interactions, and providing elevated design offerings in the store, in-home and virtually with customers",
                    "Create elevated designs for customers using the preferred design tools to create moodboards, 2D & 3D floor plans, product lists and customer presentations",
                    "Lead design consultations in person (in-store or in-home) or via email, phone and virtual",
                    "Deliver projects in a timely manner and within determined timelines",
                    "Possess a clear understanding of the brand aesthetics and merchandising strategy by channel; Store, E-Commerce, Catalog",
                    "Ensure full understanding and awareness of all product information, including characteristics, care information and staying informed with the competition and industry trends",
                    "Deliver individual sales, KPI and service goals, productivity standards, and engage customers on the sales floor by demonstrating our selling skills",
                    "Actively listen to the customer to identify which products will best meet their needs and communicate company loyalty services",
                    "(e.g. designer rewards, Design Trade Program, credit card etc)",
                    "Support and model excellent service by exhibiting a positive attitude and enthusiasm ensuring all customers are provided gracious, quick, and efficient service",
                    "Support store training and educating on design services, to drive a clear understanding of design services and offerings",
                    "Develop new and lasting relationships with customers through networking and clienteling",
                    "Understanding of basic design functions including spatial planning, fabric selection, lighting, interior design styles"
                ]
                }
            ],
            "apply_options": [
                {
                "title": "Jobs And Careers At CRATE & BARREL",
                "link": "https://jobs.crateandbarrel.com/job/natick/designer/351/90551819568?utm_campaign=google_jobs_apply&utm_source=google_jobs_apply&utm_medium=organic"
                },
                {
                "title": "ZipRecruiter",
                "link": "https://www.ziprecruiter.com/c/Crate-&-Barrel-Holdings/Job/Designer/-in-Natick,MA?jid=22904e6c0b4e6657&utm_campaign=google_jobs_apply&utm_source=google_jobs_apply&utm_medium=organic"
                },
                {
                "title": "Indeed",
                "link": "https://www.indeed.com/viewjob?jk=9f067ad88e9e7041&utm_campaign=google_jobs_apply&utm_source=google_jobs_apply&utm_medium=organic"
                },
                {
                "title": "Glassdoor",
                "link": "https://www.glassdoor.com/job-listing/designer-crate-and-barrel-JV_IC1154630_KO0,8_KE9,25.htm?jl=1009996427220&utm_campaign=google_jobs_apply&utm_source=google_jobs_apply&utm_medium=organic"
                },
                {
                "title": "Lensa",
                "link": "https://lensa.com/job-v1/crate-barrel-holdings/natick-ma/designer/7b312b191a51d6fc2c3a6fe1e537e1be?utm_campaign=google_jobs_apply&utm_source=google_jobs_apply&utm_medium=organic"
                },
                {
                "title": "SimplyHired",
                "link": "https://www.simplyhired.com/job/7y7tFfrXxx0-41KAZNwJy2ITfSqqkmRFy0ke8snWkCJcrnSVaPkc7Q?utm_campaign=google_jobs_apply&utm_source=google_jobs_apply&utm_medium=organic"
                }
            ],
            "job_id": "eyJqb2JfdGl0bGUiOiJEZXNpZ25lciIsImNvbXBhbnlfbmFtZSI6IkNSQVRFIFx1MDAyNiBCQVJSRUwiLCJhZGRyZXNzX2NpdHkiOiJOYXRpY2ssIE1BIiwiaHRpZG9jaWQiOiJraWNzUHNzbmJBUER3LWNfQUFBQUFBPT0iLCJ1dWxlIjoidytDQUlRSUNJaE1ESXhORFFzVFdGemMyRmphSFZ6WlhSMGN5eFZibWwwWldRZ1UzUmhkR1Z6IiwiZ2wiOiJ1cyIsImhsIjoiZW4ifQ=="
            },
            
        ]
    return placeholder
```

We'll want to leave this at the very bottom of our app.py.

Now, below your `index()` function, add a POST method:
```
@app.route("/search", methods=["POST"])
def search():
    return jsonify(get_placeholder())
```

[more basic JS stuff on the frontend]

So we can see that our site is displaying data properly when received in the expected format.

Now let's change `search()` to  take our inputs
```
    data      = request.get_json()
    titles    = data.get("title", "")
    location  = data.get("location", "").strip()
    work_type = data.get("work_type", "any")
    salary    = data.get("salary", "").strip()

    if not titles:
        return jsonify({"error": "Please enter at least one job title."}), 400

    all_jobs=[]
    try:
    jobs=get_placeholder()
    all_jobs.extend(jobs)
    except Exception as e:
        print(f"Error searching for '{title}': {e}")
        import traceback; traceback.print_exc()
    return jsonify({"jobs": all_jobs})

```

Normally, we'd want to just return directly from the `try` but we're going to want to build out the frontend to take multiple jobs as input. We're also going to make some modifications that allow us to retrieve cached results rather than making API requests every time.

Run your site locally to confirm it works as expected: python app.py

#### Get real data
Now to the exciting part: actually requesting (and displaying) real data. We want to have a new function, search_jobs, that retrieves our jobs data using SerpAPI.

```
def search_jobs(title: str, location: str, work_type: str, salary: str) -> list[dict]:
    """
    Return a list of job dicts. Each dict should have:
        title       str   – job title
        company     str   – company name
        location    str   – where the role is based
        work_type   str   – Remote / Hybrid / On-site
        salary      str   – salary range (or "" if unknown)
        url         str   – link to the full application
        summary     str   – short description / snippet
        posted      str   – posting date (e.g. "2 days ago")
    """

    client = serpapi.Client(api_key=API_KEY)
    results = client.search({
        "engine": "google_jobs",
        "q": title,
        "location": location,
        "google_domain": "google.com",
        "hl": "en",
        "gl": "us",
    })
    
    response = []

    for result in results["jobs_results"]:
        extended = get_extended_details(result, ["salary", "schedule_type", "posted_at", "dental_coverage", "health_insurance"])
        response.append({
            "title":     result.get("title", ""),
            "company":   result.get("company_name", ""),
            "location":  result.get("location", ""),
            "work_type": extended.get("schedule_type", ""),
            "url":       result.get("source_link", ""),
            "salary":    extended.get("salary", ""),
            "posted":    extended.get("posted_at", ""),
        })
    return response
```

If you aren't familiar with this syntax in Python, it specifies the types of our inputs and outputs rather than allowing us to pass whatever we want to the function.

We also want a helper function, get_extended_results, that will get details like salary, schedule type (full-time, part-time, internship, etc), when the job was posted, and relevant benefits:

```
def get_extended_details(result, details_list):
  
    details_output = {}
    for detail_field in details_list:
        try:
            details_output[detail_field] = result["detected_extensions"][detail_field]
        except:
            details_output[detail_field] = "See full listing"

    
    return details_output
```

We use this syntax so that we can gracefully handle if one or more of the requested fields is missing.

## Search for multiple job titles in one search using the tag input design pattern

A little background: My motivation in building this little app was that I hate job searching because sometimes, I'm curious about several different types of roles. Let's say I'm really looking for a full-time developer relations role (true), but I'm also open to temporary part-time work as a retail stylist while I'm doing contract work in e-commerce management (also true). I don't want to run three searches every time I want to see what's out there - I want to see all the roles together and choose for myself how to prioritize them.

Anyway, to solve for this, I added the tag input design pattern so you can search for several types of roles at once.

### Changes to app.py

Modify search() to work for several roles:
```
@app.route("/search", methods=["POST"])
def search():
    data      = request.get_json()
    titles    = data.get("titles", [])
    location  = data.get("location", "").strip()
    work_type = data.get("work_type", "any")
    salary    = data.get("salary", "").strip()

    if not titles:
        return jsonify({"error": "Please enter at least one job title."}), 400

    all_jobs = []
    for title in titles:
        try:
            jobs = search_jobs(title, location, work_type, salary)
            all_jobs.extend(jobs)
        except Exception as e:
            print(f"Error searching for '{title}': {e}")
            import traceback; traceback.print_exc()


    return jsonify({"jobs": all_jobs})
```

We want to iterate over each job entered in the client and compile all results so we can display all the results together.

### Changes to index.html
And we have a handful of changes to make to index.html so we can collect inputs as tags, separated by either a comma or the enter key, displayed as tags (the vision here was Google Flights' multiple destination search option).

We run the app again locally to make sure things work as expected

## Cache results

SerpAPI has monthly usage limits, and I'm trying to avoid hitting them, so let's write our search responses to a database for reference when searching. Let's say we only need fresh data for a search method every 24 hours.

Create a new file db.py:
```
import os
import json
import hashlib
import psycopg2
from datetime import datetime, timedelta, timezone

DATABASE_URL=os.getenv("DATABASE_URL")

def get_conn():
    try:
        return psycopg2.connect(DATABASE_URL, sslmode="require")
    except Exception as e:
        print(e)
        print(DATABASE_URL)
        return
        
def make_key(description: str, location: str, work_type: str, salary: str) -> str:
    raw = f"{description}|{location}|{work_type}|{salary}".lower().strip()
    return hashlib.sha256(raw.encode()).hexdigest()

def get_cached(cache_key: str) -> list | None:
    cutoff = datetime.now(timezone.utc) - timedelta(hours=24)
    with get_conn() as conn, conn.cursor() as cur:
        cur.execute("""
            SELECT results FROM search_cache
            WHERE cache_key = %s AND created_at > %s
        """, (cache_key, cutoff))
        row = cur.fetchone()
    return row[0] if row else None

def set_cached(cache_key: str, role: str, location: str, results: list):
    with get_conn() as conn, conn.cursor() as cur:
        cur.execute("""
            INSERT INTO search_cache (cache_key, role, location, results, created_at)
            VALUES (%s, %s, %s, %s, NOW())
            ON CONFLICT (cache_key) DO UPDATE
                SET results = EXCLUDED.results,
                    created_at = NOW()
        """, (cache_key, role, location, json.dumps(results)))
```

Now we have a handful of functions at our disposal. We can create a connection using the database URL in the user's bash profile, set a search response to be cached, and retrieve cached results. To use these functions, we'll return to app.py and edit the for loop in `search()`:
```
    for title in titles:
        cache_key = make_key(title, location, work_type, salary)
        cached = get_cached(cache_key)
        try:
            if cached:
                all_jobs.extend(cached)
            else:
                jobs = search_jobs(title, location, work_type, salary)
                set_cached(cache_key, title, location, jobs)
                all_jobs.extend(jobs)
        except Exception as e:
            print(f"Error searching for '{title}': {e}")
            import traceback; traceback.print_exc()
            
```

So now we're retrieving the results if they exist in the cache and aren't outdated, otherwise we run a search and cache the results for next time.

# Customizations I implemented
## Daytime mode
As I already admitted, I didn't write our frontend code because I didn't want to. The default styling was pretty dark for me. I wanted a Daytime/Nighttime mode so users can toggle and choose a layout for themselves.
```
```

## Loading indication
The app doesn't return results instantaneously, so it's nice to show the user that you're working on their request. 
In our web client, at the very top of the search form, just inside the `<body>` tag, we add the following (I used the sun emoji as the loading spinner because every job search can use the help of a little sunshine):

```
<div class="loading-overlay" id="loadingOverlay">
  <div class="loading-sun">☀️</div>
  <div class="loading-text">Searching...</div>
</div>
```

Under Helpers, we add
```
function setLoading(on) {
  const btn = document.getElementById('searchBtn');
  const overlay = document.getElementById('loadingOverlay');

  btn.disabled = on;
  overlay.classList.toggle('visible', on);

  if (on) {
    document.getElementById('jobList').innerHTML = '<div class="list-state">Searching...</div>';
    document.getElementById('resultCount').textContent = '';
  }
}
```

and under `clearError();` in doSearch (before we fetch data), we add `setLoading(true);`
At the very end of doSearch, we add a finally block to `setLoading(false);`

We also need to set styling for the overlay. In `style.css`:

```
/* empty / loading / error states */
.list-state {
  padding: 28px 16px;
  font-size: 12px;
  color: var(--subtext);
  line-height: 1.7;
  white-space: nowrap;
  overflow: hidden;
}

.spinner {
  display: inline-block;
  width: 14px; height: 14px;
  border: 2px solid var(--muted);
  border-top-color: var(--accent);
  border-radius: 50%;
  animation: spin .7s linear infinite;
  vertical-align: middle;
  margin-right: 8px;
}
@keyframes spin { to { transform: rotate(360deg); } }

.loading-overlay {
  display: none;
  position: fixed;
  inset: 0;
  background: rgba(253, 240, 245, 0.75);
  backdrop-filter: blur(3px);
  z-index: 100;
  align-items: center;
  justify-content: center;
  flex-direction: column;
  gap: 12px;
}

.loading-overlay.visible {
  display: flex;
}

.loading-sun {
  font-size: 48px;
  animation: spin 1.5s linear infinite;
}

.loading-text {
  font-family: var(--font-head);
  font-size: 13px;
  font-weight: 600;
  letter-spacing: 0.12em;
  text-transform: uppercase;
  color: var(--subtext);
}
```

# Deploying our app online

First things first, we don't want to use our own API access here, so [our app]("https://hamcclellan.pythonanywhere.com/") will return sample data instead.

I'm hosting on [PythonAnywhere](https://www.pythonanywhere.com/) (frankly, because I've never used it and was curious). There's an option after you walk through the basic tour for hosting an app you've implemented locally. First, I realized I want to have fallback for our API key and database URL, since those are going to be empty here. I modified search() and search_jobs() in app.py to allow for both environment variables to be None, then I proceeded implementing the hosting setup.

See the live app [here](https://hamcclellan.pythonanywhere.com/)

# Future improvements

- Adding salary and role types as search constraints
  - The SerpAPI client search is somewhat bare - role type (hybrid/remote/in-person) and schedule type (full-time/part-time/internship) aren't their own parts of the query. We could refactor the search_jobs method to return objects and create additional Search methods to further filter results to meet those requirements.

- Displaying results of several more specific searches in one place
  - Just as I want to see several types of roles, it's possible a user would want to see roles in multiple different locations conditional on the role itself. For an example, a college student might be open to a summer internship in one of five cities, or willing to work seasonal retail near their college town or hometown, but not be willing to relocate for a seasonal retail job.

- Deduplicating roles
  - If I'm looking for project management roles and e-commerce management roles, it's likely both searches will return some number of overlapping roles. We can use the job ID field of the responses as, well, unique identifiers for each result.

- Night mode loading spinner
  - Finally, a fun little addition: set the spinner for the Loading indicator to be a moon emoji if the page is in Night Mode.

[repo-link]: https://github.com/harchermcclellan/job-finder
[limited-live-demo]: https://hamcclellan.pythonanywhere.com/
