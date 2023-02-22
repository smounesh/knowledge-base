## Run a cmd in every dir

1. Run a cmd inside every dir of /opt if docker-compose.yml file is present in that dir
```bash
for dir in /opt/*/; do
  echo "Checking directory: ${dir}"
  if [ -f "${dir}docker-compose.yml" ]; then
    echo "Found docker-compose.yml file in ${dir}"
    docker-compose up -d;
    cd "${dir}" && docker-compose up -d
    echo "Ran command in ${dir}"
  else
    echo "No docker-compose.yml file found in ${dir}"
  fi
done
```