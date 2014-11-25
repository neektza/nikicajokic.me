task :deploy do
  sh 'bundle exec jekyll build && rsync -r _site root@floatingpoint.io:/home/www/pltconfusion.com'
end
