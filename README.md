## Introduction

[tech-lessons.in](https://tech-lessons.in/) is a [Hugo](https://gohugo.io/) based blog and this repository the core content of the blog.

## Running on local

    # Clone the repository
    git clone https://github.com/SarthakMakhija/tech-lessons-posts.git        

    # Assuming npm is installed, install the dependencies
    npm i

    # Start the server
    npm start

## Publish to Github Pages

    # Creates a local gh-pages branch which will contain the public folder
    git checkout -b gh-pages

    # Build the website
    npm i && HUGO_ENVIRONMENT=production hugo --gc --minify

    # Change to public/ directory
    cd public
    
    # Initialize the git repository
    git init
    git add .
    git commit -m "Website build"

    # Add the remote
    git remote add website git@github.com:SarthakMakhija/sarthakmakhija.github.io.git

    # Push the content
    git push website main --force
    
    # May need to provide the custom domain 
    Provide the custom domain [here](https://github.com/SarthakMakhija/sarthakmakhija.github.io/settings/pages)


## Website repository

+ [tech lessons theme repository](https://github.com/SarthakMakhija/tech-lessons-hugo-theme)
  + This repository is forked from the [blist](https://github.com/apvarun/blist-hugo-theme) Hugo theme
+ [tech lessons repository](https://github.com/SarthakMakhija/sarthakmakhija.github.io)
