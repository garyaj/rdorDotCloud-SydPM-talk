
### How to deploy a RDOR web app on DotCloud.com

[DotCloud(DC)](http://www.dotcloud.com/) has installed a Perl/PSGI/Nginx stack for Perl apps (they call them services) and a PostgreSQL(Pg) service which together allow rapid deployment of Rose::DBx::Object::Renderer (RDOR) web apps in the cloud. This article uses Danny's tutorial examples for using [Rose::DBx::Object::Builder (RDOB)](https://github.com/dannyglue/Rose-DBx-Object-Renderer/wiki/Generate-database-tables-with-rose%3A%3Adbx%3A%3Aobject%3A%3Abuilder) and [Rose::DBx::Object::Renderer with CGI::Application](https://github.com/dannyglue/Rose-DBx-Object-Renderer/wiki/Integrating-with-CGI%3A%3AApplication).


On DotCloud's tutorial page for Perl they note:
    "The perl service can host any Perl web application compatible with the PSGI standard."


CGI::Application from version 4.5 onward supports PSGI so it is relatively
painless to port a CGI::App to DotCloud. Here's what I did.


I recommend following the DotCloud [First Steps Tutorial](http://docs.dotcloud.com/tutorials/firststeps/) first to familiarise yourself with the tools and terms used.

#### Overview

To deploy an RDOR app to DotCloud:

1. use RDOB to deploy a PostgreSQL database as a service on DotCloud;
2. write or modify an RDOR application to use CGI::Application;
3. make the application runnable locally with plackup;
4. write a Makefile.PL listing all the high-level dependencies;
5. deploy the app to DotCloud;
6. access the app with your browser.

#### Deploying the database

1. Sign up to get a Beta account invitation to DotCloud. (Used to take days but usually within the hour now.)
2. I recommend following Danny's tutorial [Generate Database Tables with Rose::DBx::Object::Builder](https://github.com/dannyglue/Rose-DBx-Object-Renderer/wiki/Generate-database-tables-with-rose%3A%3Adbx%3A%3Aobject%3A%3Abuilder) to create a Perl app which will create your desired Pg database schema on a locally hosted instance of Pg. Then you can run the app with your DotCloud credentials and it will create your Pg DotCloud service ready to be used by other apps. Or output the Rose::DBx::Object::Builder code to a text file (or hand construct the schema with a text editor (too much work for me :) ) which you can install using psql and your DC credentials from the commandline). Note: if you name the database 'application' instead of 'demo' you will create a database which will connect to the RDOR app with no editing required. 
3. Follow the [First Steps Tutorial](http://docs.dotcloud.com/tutorials/firststeps/) at least upto the beginning of 'Deploy your first service'
i.e. 
    1. Signup with invitation code and create DC account;
    2. Assuming you have >= Python 2.6 installed, install dotcloud commandline tool:

            sudo easy_install dotcloud

    3. Type 'dotcloud' and enter API key from DC account 'Settings' page;
    4. Create your app namespace (I chose 'nasi' instead of the 'ramen' used in the First Steps tutorial; Pg tutorial uses 'louis');
    1. Deploy a Pg service:

            dotcloud deploy -t postgresql nasi.sql
      
    2. Create user 'demo':
      
            dotcloud run nasi.sql -- createuser demo -SDRP

    3. Create database 'application':
        
            dotcloud run nasi.sql createdb application
      
    4. Give all rights on database 'application' to user 'demo':
        
            dotcloud run nasi.sql -- psql -c \"GRANT ALL PRIVILEGES ON DATABASE application TO demo\"
      
    5. Get the uri and port of your Pg service:
      
            $ dotcloud info nasi.sql
            cluster: wolverine
            config: postgresql_password: 'password'
            created_at: 1304204973.0931809
            name: nasi.sql
            namespace: nasi
            ports:
            name: sql
              url:
              pgsql://root:password@sql.nasi.dotcloud.com:3317
            name: ssh
              url: ssh://postgres@sql.nasi.dotcloud.com:3108
            state: running
            type: postgresql

    6. Modify the config line of the Builder app you created in the RDOB tutorial above to use the uri and port in the sql url above (i.e. `sql.nasi.dotcloud.com` and `3317`) (yours will be different of course):

            my $builder = Rose::DBx::Object::Builder->new(
              config => {
                db => {
                  type => 'Pg',
                  name => 'application', 
                  host => 'sql.nasi.dotcloud.com',
                  port => 3317,
                  username => 'demo',
                  password =>'password',
                  tables_are_singular => 1,
                }
              }
            );

    7. Run the app to create your schema on DotCloud:
      
            perl bld_appdb.pl

    8. Check the 'application' tables:

            $ dotcloud run nasi.sql -- psql application
            # psql application
            psql (9.0.3)
            Type "help" for help.

            application=# \dt
                          List of relations
            Schema |         Name         | Type  | Owner 
            --------+----------------------+-------+-------
            public | employee             | table | root
            public | employee_project_map | table | root
            public | position             | table | root
            public | project              | table | root
            (4 rows)

            application=# \q

        Successfully deployed!

#### Deploying the RDOR app.

  1. As noted in [DotCloud's tutorial page for Perl](http://docs.dotcloud.com/components/perl/), create a directory to build the app e.g. rdor.
  2. Only three files are essential: app.psgi, Makefile.PL and lib/Application.pm.
    1. app.psgi loads the Application.pm module and runs it under PSGI;
    2. Makefile.PL lists the dependencies of the app and DC runs cpanm to import all the dependencies to the stack prior to running the app;
    3. Application.pm is main body of code used in the app.
  1. I copied Danny's [RDOR+CGI::App](https://github.com/dannyglue/Rose-DBx-Object-Renderer/wiki/Integrating-with-CGI%3A%3AApplication) tutorial code into the rdor directory but discovered, by trial and error that I needed to make one small change. I changed the line which read:

        sub start : Start {
    
    to
    
        sub list : StartRunmode {

    There is no 'Start' mode in AutoRunmode, the correct mode is 'StartRunmode' and the double 'start' seems to confuse C::A::PSGI.
    Be aware that you will have to change the RDOR config line of Application.pm when switching between running locally with plackup or running on DotCloud:
    For (local) plackup:

        type => 'Pg',
        name => 'application',
        username => 'demo',
        password => '',

    For DotCloud deployment (as for RDOB above):

        type => 'Pg',
        name => 'application',
        host => 'sql.nasi.dotcloud.com',
        port => 3317,
        username => 'demo',
        password =>'password',

    Put Application.pm in the rdor/lib directory.
  2. Create app.psgi:

        use CGI::PSGI;
        use lib './lib';
        use Application;

        my $handler = sub {
          my $env = shift;
          my $app = Application->new({ QUERY => CGI::PSGI->new($env) });
          CGI::Application::PSGI->run($app);
        };

    This is very similar to the CGI and FCGI examples Danny has included in the tutorial.
  3. Run app.psgi with plackup to test the script locally:

        $ plackup app.psgi
        HTTP::Server::PSGI: Accepting connections at http://0:5000/

    If your typing/copying is good and all the required CPAN modules are on your local system you will be able see the default start page of the app in your browser at '`http://0:5000`'.
  4. Create a Makefile.PL listing the high-level dependencies (DotCloud's Perl stack includes all the standard Perl modules):

        use ExtUtils::MakeMaker;
        WriteMakefile(
          PREREQ_PM => {
            'CGI::Application'                         => 0,
            'CGI::Application::Plugin::AutoRunmode'    => 0,
            'CGI::Application::Plugin::Authentication' => 0,
            'DBD::Pg'                                  => 0,
            'Rose::DBx::Object::Renderer'              => 0,
            'CGI::PSGI'                                => 0,
            'Plack::Request'                           => 0,
          }
        );

    A couple of notes of explanation. DotCloud's call to cpanm to install Rose::DBx::Object::Renderer will also install all the Rose::DB dependencies except the database driver so it is necessary to include DBD::Pg. The need to include Plack::Request for CGI::Application::PSGI was discovered by examining the logs after app crashed.
  5. Deploy a Perl stack on DotCloud:

        dotcloud deploy -t perl nasi.rdor

  6. Push your app to this stack:

        dotcloud push nasi.rdor .

    Note the '.' at the end meaning 'this directory'. The first time you push you'll see a long list of CPAN modules being installed into deployment. Eventually you should see the following:

        <== Installed dependencies for .. Finishing.
        uwsgi: stopped
        uwsgi: started
        Connection to rdor.nasi.dotcloud.com closed.

  7. Browse to your app:

        http://rdor.nasi.dotcloud.com

    If you get the default page of your app, you've succeeded!
  8. **However** there's all sorts of problems which can occur and they're not always obvious. The error I keep making is switching the RDOR config line between local and DC. That will give you a 404 error. Another frequently occuring initial problem is a dependency missing. Have a look at the logs using:

        dotcloud logs nasi.sql

    This command runs a '`tail -F`' on the logs as well so you can monitor in realtime.
  9. Now it's time to make your app look prettier with templates and styling. Just edit locally, check with perl -wc and plackup then push to DotCloud. Simple!

#### Conclusion

There's a lot to understand at first and there's a bit too much trial and error required but once the initial setup is working updates are simple and DotCloud seems pretty reliable. Eventually you will need to read all the DotCloud documentation but hopefully this will get you started with Rose::DBx::Object::Renderer on DotCloud.
