[TOC]

# Text Files

## concatenate and omit the first line of every file

```bash
for filename in *; do 
	cat "$filename" | tail -n +2 >> out.csv;
done
```

