# Run an ephemeral container to test locally 
docker run -rm --volume="$PWD:/srv/jekyll:Z" --publish 4000:4000 jekyll/jekyll jekyll serve --drafts
