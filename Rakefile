namespace :assets do
  desc 'Precompile assets'
  task :precompile do
    sh "bundle exec jekyll build"
    sh "bundle exec compass compile ./res"
  end
end
