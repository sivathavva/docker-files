# Command to fetch docker tags  with Pagination
NEXT_URL="https://hub.docker.com/v2/repositories/sivathavva/python-app/tags/"
while [ "$NEXT_URL" != "null" ]; do
  RESPONSE=$(curl -s "$NEXT_URL")
  echo "$RESPONSE" | jq -r '.results[].name'
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

_______________________________________________________________________________________________________________________________________________________________
________________________________________________________________________________________________________________________________________________________________
#Delete docker tags if count exceeds 10

name: Fetch, Process, and Manage Docker Hub Tags

on:
  workflow_dispatch: # Allows manual triggering of the workflow

jobs:
  fetch-and-manage-tags:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout Repository
      uses: actions/checkout@v3

    - name: Set up jq
      run: sudo apt-get install -y jq

    - name: Authenticate with Docker Hub (Bearer Token)
      env:
        DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
        DOCKER_TOKEN: ${{ secrets.DOCKER_TOKEN }}
      run: |
        echo "$DOCKER_TOKEN" | docker login -u "$DOCKER_USERNAME" --password-stdin

    - name: Fetch, Process, and Manage Docker Hub Tags
      env:
        DOCKER_REPO: ${{ secrets.DOCKER_REPO }}
        DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
        DOCKER_TOKEN: ${{ secrets.DOCKER_TOKEN }}
      run: |
        NEXT_URL="https://hub.docker.com/v2/repositories/${DOCKER_REPO}/tags/"
        > docker_tags_numbers.txt # Clear or create the file for numeric tags
        while [ "$NEXT_URL" != "null" ]; do
          RESPONSE=$(curl -u "$DOCKER_USERNAME:$DOCKER_TOKEN" -s "$NEXT_URL")
          
          # Filter tags with numeric values and append them to the file
          echo "$RESPONSE" | jq -r '.results[].name' | grep -E '^[0-9]+$' >> docker_tags_numbers.txt
          
          NEXT_URL=$(echo "$RESPONSE" | jq -r '.next')
        done

        # Sort numeric tags in descending order and keep the top 10
        SORTED_TAGS=($(sort -nr docker_tags_numbers.txt))
        KEEP_TAGS=(${SORTED_TAGS[@]:0:10})
        echo "Tags to keep: ${KEEP_TAGS[@]}"

        # Identify and delete tags not in the top 10
        for TAG in "${SORTED_TAGS[@]}"; do
          if [[ ! " ${KEEP_TAGS[@]} " =~ " ${TAG} " ]]; then
            echo "Deleting tag: $TAG"
            curl -X DELETE "https://hub.docker.com/v2/repositories/${DOCKER_REPO}/tags/$TAG/" -H "Authorization: Bearer $DOCKER_TOKEN"
          fi
        done

        echo "Tag cleanup completed."
