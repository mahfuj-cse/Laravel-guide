name: Run Tests

on:
  pull_request:
    branches: [ main, test ]
    paths:
      - 'tests/**'
      - 'app/**'
      - 'composer.json'
      - 'composer.lock'
  push:
    branches: [ main, test ]
    paths:
      - 'tests/**'
      - 'app/**'
      - 'composer.json'
      - 'composer.lock'

jobs:
  test:
    runs-on: ubuntu-latest
    # This job name will be used as required status check
    
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: password
          MYSQL_DATABASE: testing
        ports:
          - 3306:3306
        options: --health-cmd="mysqladmin ping" --health-interval=10s --health-timeout=5s --health-retries=3

    steps:
    - uses: actions/checkout@v3

    - name: Setup PHP
      uses: shivammathur/setup-php@v2
      with:
        php-version: '8.1'
        extensions: mbstring, dom, fileinfo, mysql, gd
        coverage: xdebug

    - name: Install Dependencies
      run: composer install -q --no-ansi --no-interaction --no-scripts --no-progress --prefer-dist

    - name: Set Environment Variables
      run: |
        echo "APP_KEY=${{ secrets.APP_KEY }}" >> $GITHUB_ENV
        echo "AWS_ACCESS_KEY_ID=${{ secrets.AWS_ACCESS_KEY_ID }}" >> $GITHUB_ENV
        echo "AWS_SECRET_ACCESS_KEY=${{ secrets.AWS_SECRET_ACCESS_KEY }}" >> $GITHUB_ENV
        echo "AWS_DEFAULT_REGION=ap-northeast-1" >> $GITHUB_ENV
        echo "AWS_COGNITO_ACCESS_KEY_ID=${{ secrets.AWS_COGNITO_ACCESS_KEY_ID }}" >> $GITHUB_ENV
        echo "AWS_COGNITO_SECRET_ACCESS_KEY=${{ secrets.AWS_COGNITO_SECRET_ACCESS_KEY }}" >> $GITHUB_ENV
        echo "AWS_COGNITO_REGION=ap-northeast-1" >> $GITHUB_ENV
        echo "AWS_COGNITO_APP_CLIENT_ID=${{ secrets.AWS_COGNITO_APP_CLIENT_ID }}" >> $GITHUB_ENV
        echo "AWS_COGNITO_APP_CLIENT_SECRET=${{ secrets.AWS_COGNITO_APP_CLIENT_SECRET }}" >> $GITHUB_ENV
        echo "AWS_COGNITO_USER_POOL_ID=${{ secrets.AWS_COGNITO_USER_POOL_ID }}" >> $GITHUB_ENV
        echo "TWILIO_SID=${{ secrets.TWILIO_SID }}" >> $GITHUB_ENV
        echo "TWILIO_TOKEN=${{ secrets.TWILIO_TOKEN }}" >> $GITHUB_ENV
        echo "MUX_TOKEN_SECRET=${{ secrets.MUX_TOKEN_SECRET }}" >> $GITHUB_ENV
        echo "SLACK_WEBHOOK_URL=${{ secrets.SLACK_WEBHOOK_URL }}" >> $GITHUB_ENV

    - name: Directory Permissions
      run: chmod -R 777 storage bootstrap/cache

    - name: Run Database Migrations
      env:
        DB_CONNECTION: mysql
        DB_HOST: 127.0.0.1
        DB_PORT: 3306
        DB_DATABASE: testing
        DB_USERNAME: root
        DB_PASSWORD: password
      run: php artisan migrate --force

    - name: Get changed test files
      id: changed-files
      uses: tj-actions/changed-files@v39
      with:
        files: tests/**/*.php

    - name: Run Tests
      env:
        DB_CONNECTION: mysql
        DB_HOST: 127.0.0.1
        DB_PORT: 3306
        DB_DATABASE: testing
        DB_USERNAME: root
        DB_PASSWORD: password
      run: |
        if [ "${{ steps.changed-files.outputs.any_changed }}" == "true" ]; then
          echo "Running tests for changed files:"
          echo "${{ steps.changed-files.outputs.all_changed_files }}"
          TEST_FAILED=0
          for file in ${{ steps.changed-files.outputs.all_changed_files }}; do
            if ! php artisan test "$file"; then
              TEST_FAILED=1
            fi
          done
          if [ $TEST_FAILED -eq 1 ]; then
            echo "❌ Tests failed! PR cannot be merged."
            exit 1
          fi
          echo "✅ All tests passed!"
        else
          echo "No test files changed, skipping tests"
        fi