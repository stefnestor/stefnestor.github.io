# stefnestor.github.io 

`/remind` 
- create new 
	```bash
	$ hugo new content labs/bash-report-append.md
	```
- new post metadata formats
	```markdown
	+++
	title = 'Elasticsearch Outline'
	date = 2022-04-03T12:00:00-07:00
	draft = false
	type = 'post'
	showTableOfContents = true
	tags = [ 'elastic', 'elasticsearch', 'jq' ]
	+++
	```
- local compile (before `git push` for automated Github Actions to publish)
	```bash
	$ hugo -F
	```
