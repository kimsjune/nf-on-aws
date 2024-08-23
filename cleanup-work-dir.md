### Background

Need to minimize the storage cost on s3 by deleting `work` directories that are not relevant to the most recent, successful Nextflow job. `nextflow clean` might do the trick if the logs are simple and more recent runs don't depend too much on cached data. But if I `clean` too deep, there is a risk of deleting completed tasks that have to be run again redundantly. This is a balancing act between storage and compute.

### Directories to keep

`nextflow log last > log_dir.txt`

### Script

```bash
# aws cli to ls all directories recursively
aws s3 ls s3://jk-rnaseq/work --recursive > all_dir.txt

# remove the first line which is the parent directory work/ (obviously don't want to delete)
tail -n +2 all_dir.txt > all_dir.txt

# extract the relative paths (last column), keep paths up to the 3rd /, append bucket name, and save unique lines
cat all_dir.txt | awk '{print $NF}' | sed 's/\/[^/]*//3g' | sed 's/^/s3:\/\/jk-rnaseq\//' | uniq  > all_dir_uniq.txt

# directories that I want to delete are unique to all_dir_uniq.txt
cat log_dir.txt all_dir_uniq.txt | sort | uniq -u > to_delete.txt

# use xargs to iterative command over each line
cat to_delete.txt | xargs -r -P 1 -n 1 aws s3 rm--recursive
```
### Eureka moment  

`uniq` merges matching lines to the first occurrence.  
`uniq -u` only prints unique, unduplicated lines.

### Aftermath

`work` directory size decreased about ~25% of original

### Notes

Referenced [this](https://gist.github.com/photocyte/495848faaba3319c962a575593eaeb55) and [#4308](https://github.com/nextflow-io/nextflow/discussions/4308). 

