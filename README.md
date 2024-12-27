# Command to fetch docker tags 
NEXT_URL="https://hub.docker.com/v2/repositories/sivathavva/python-app/tags/"
while [ "$NEXT_URL" != "null" ]; do
    # Fetch the current page of results
  RESPONSE=$(curl -s "$NEXT_URL")
  
      # Extract and print tag names from the current page
  echo "$RESPONSE" | jq -r '.results[].name'
  
     # Update NEXT_URL to the next page's URL
  NEXT_URL=$(echo "$RESPONSE" | jq -r '.next')
done

________________________________________________________________________________________________________________________________________
________________________________________________________________________________________________________________________________________
# Github actions yaml file to fetch tags 
name: Fetch Docker Hub Tags

on:
  workflow_dispatch: # Allows manual triggering of the workflow

jobs:
  fetch-tags:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout Repository
      uses: actions/checkout@v3

    - name: Set up jq
      run: sudo apt-get install -y jq

    - name: Authenticate with Docker Hub
      env:
        DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
        DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
      run: |
        echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin

    - name: Fetch Docker Hub Tags
      env:
        DOCKER_REPO: ${{ secrets.DOCKER_REPO }}
      run: |
        NEXT_URL="https://hub.docker.com/v2/repositories/${DOCKER_REPO}/tags/"
        > docker_tags.txt # Clear or create the file
        while [ "$NEXT_URL" != "null" ]; do
          RESPONSE=$(curl -u "$DOCKER_USERNAME:$DOCKER_PASSWORD" -s "$NEXT_URL")
          echo "$RESPONSE" | jq -r '.results[].name' >> docker_tags.txt
          NEXT_URL=$(echo "$RESPONSE" | jq -r '.next')
        done
        cat docker_tags.txt # Optional: Print file content
