1. Configure the Heroku Command Line Interface (CLI) using these steps: [Heroku CLI](https://devcenter.heroku.com/articles/heroku-cli).

1. Deploy the Standard File server to your account using this deploy link: https://heroku.com/deploy?template=https://github.com/standardfile/ruby-server.

1. Install a MySQL addon. Here we'll use the JawsDB addon: https://elements.heroku.com/addons/jawsdb. If you already have a database, you can skip this step.

1. After installing JawsDB, choose it from the addons list in your Heroku dashboard. This will take you to your JawsDB dashboard. You'll need the information found here for the next step.

1. In your Heroku application, choose the "Settings" tab.

1. In the "Config Variables" section, click "Reveal Config Variables".

1. Add 4 new variables:

	- `DB_HOST`: Use the "Host" value from your JawsDB dashboard.
	- `DB_DATABASE`: In your JawsDB dashboard, you should see at the top "Connection String". Copy the part after "3306/". That will be the name of your database.
	- `DB_USERNAME`: Use the "Username" value.
	- `DB_PASSWORD`: Use the "Password" value.

1. Restart your Heroku instance using either the web interface (you'll find this option under "More" in the top right) or using the command line:

	```
	heroku restart --app name_of_app
	```

1. Perform initial database setup:

	```
	heroku run rake db:migrate --app name_of_app
	```

And that's it! You're up and running with a free Standard File server that you can use in [Standard Notes](https://standardnotes.org).

Note that you should understand the limitations of Heroku's free tier. In particular, your instance will sleep after 30 minutes of idleness, and may take several seconds to start up again on subsequent requests.
