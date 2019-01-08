---
published: false
---
## A New Post

Enter text in [Markdown](http://daringfireball.net/projects/markdown/). Use the toolbar above, or click the **?** button for formatting help.

Recently I ran into situation where I needed to access the Certificate Store *scoped* to a specific Windows Service.  If you're not familiar with this, the Certificate Store on Windows can be scoped to different levels.  There's the global store that's scoped to `LocalMachine`, then there are user-specific stores which default to the `CurrentUser`.

Additionally, since at least Windows 2008, Certificate Store access can be scoped to a specific Windows Service.  This can be useful if you want to compartmentalize and isolate certain certificates to certain services.  For some services, this may even enable certain unique features, which is my use case that brought me here.

In this instance, I was 