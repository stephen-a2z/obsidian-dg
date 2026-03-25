---
{"dg-publish":true,"permalink":"/01-技术分享/vercel 和cloud flare托管 国内访问速度对比/","noteIcon":"","created":"2026-03-23T13:27:55.993+08:00","updated":"2026-03-23T14:05:14.824+08:00"}
---


这个项目本来是默认vercel托管的, 但是我自己感受了下不用代理直接国内流量访问, 表现得很一般, 于是我想着迁移到cloudflare上会不会快一些, 于是做了个对比测试. 

分别做了两个站点:  
- 托管到cloudflare 的站点 blogs.aiports.dev
- 托管到vercel 的站点 blog.aiports.dev

## 17ce.com 测试结果:

- blogs.aiports.dev - cloudflare

![Pasted image 20260323133501.png](/img/user/assets/Pasted%20image%2020260323133501.png)
![Pasted image 20260323133528.png](/img/user/assets/Pasted%20image%2020260323133528.png)

- blog.aiports.dev - vercel
![Pasted image 20260323133630.png](/img/user/assets/Pasted%20image%2020260323133630.png)


## catchpoint.com 测试结果

- blogs.aiports.dev - cloudflare
![Pasted image 20260323135015.png](/img/user/assets/Pasted%20image%2020260323135015.png)

- blog.aiports.dev - vercel

![Pasted image 20260323135936.png](/img/user/assets/Pasted%20image%2020260323135936.png)