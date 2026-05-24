# jordanaperry-blog

Hugo blog source for https://blog.jordanaperry.com

---

## Writing workflow

1. Open `jordanaperry-blog.code-workspace` in Finder
2. Create a new post in `content/posts/` — copy an existing one as a template
3. Write in markdown
4. Preview locally (optional): `hugo server` then open http://localhost:1313
5. When ready to publish:
```bash
   git add .
   git commit -m "new post: your title here"
   git push
```
6. Blog auto-deploys in a few seconds at https://blog.jordanaperry.com

---

## Stack

- Hugo v0.147.1 + PaperMod theme
- Hosted on self-hosted blog server (CT 101, twilight VLAN)
- Auto-deploy via Gitea webhook → deploy script on blog server