# TableMigrator

by Matt Freels and Rohith Ravi


Zero-downtime migrations of large tables in MySQL. 

See the example for usage. Make sure you have an index on updated_at.

Install as a rails plugin:

    ./script/plugin install git://github.com/freels/table_migrator.git


### What this does

TableMigrator is a strategy for altering large MySQL tables while incurring as little downtime as possible. There is nothing special about ALTER TABLE. All it does is create a new table with the new schema and copy over each row from the original table. Oh, and it does this in a big giant table lock, so you won't be using that big table for a while...

TableMigrator does essentially the same thing as ALTER TABLE, but more intelligently, since we can be more intelligent when we know something about the data being copied. First, we create a new table like the original one, and then apply one or more ALTER TABLE statements to the unused, empty table. Second we copy all rows from the original table into the new one. All this time, reads and writes are going to the original table, so the two tables are not consistent. 

The solution to find updated or new rows is to use updated_at (if a row is mutable) or created_at (if it is immutable) to determine which rows have been modified since the copy started. Those rows are copied over to the new table using an INSERT TABLE sequence with an ON DUPLICATE KEY clause.

If we are doing a single delta pass (you set :multi_pass to false), then a write lock is acquired before the delta copy, the delta is copied to the new table, the tables are swapped, and then the lock is released.

The :mutli_pass version does the same thing above but copies the delta in a non-blocking manner multple times until the length if time taken is small enough. Finally, the last delta copy is done synchronously, and the tables are swapped. Usually the delta left for this last copy is extremely small. Hence the claim zero-downtime migration.

The key to the multi_pass version is having an index on created_at or updated_at. Having an index on the relevant field makes looking up the delta much faster. Without that index, TableMigrator has to do a table scan while holding the table write lock, and that means you are definitely going to incur downtime.


## Example Migration

    class AddAColumnToMyGigantoTable < ActiveRecord::Migration

      # just a helper method so we don't have to repeat this in self.up and self.down
      def self.setup

        # create a new TableMigrator instance for the table `users`
        # :migration_name - a label for this migration, used to rename old tables.
        #                   this must be unique for each migration using a TableMigrator
        # :multi_pass     - copy the delta asynchronously multiple times.
        # :dry_run        - set to false to really run the tm's SQL.
        @tm = TableMigrator.new("users",
          :migration_name => 'random_column',
          :multi_pass => true,
          :dry_run => false
        )
        
        # push alter tables to schema_changes
        @tm.schema_changes.push <<-SQL
          ALTER TABLE :new_table_name 
          ADD COLUMN `foo` int(11) unsigned NOT NULL DEFAULT 0
        SQL

        @tm.schema_changes.push <<-SQL
          ALTER TABLE :new_table_name 
          ADD COLUMN `bar` varchar(255)
        SQL

        @tm.schema_changes.push <<-SQL
          ALTER TABLE :new_table_name 
          ADD INDEX `index_foo` (`foo`)
        SQL

        # for convenience
        common_cols = %w(id name session password_hash created_at updated_at)    

        # the base INSERT query with no wheres. (We'll take care of that part.)
        @tm.base_copy_query = <<-SQL
          INSERT INTO :new_table_name (#{common_cols.join(", ")}) 
          SELECT #{common_cols.join(", ")} FROM :table_name
        SQL

        # specify the ON DUPLICATE KEY update strategy.
        @tm.on_duplicate_update_map = <<-SQL
          #{common_cols.map {|c| "#{c}=VALUES(#{c})"}.join(", ")}
        SQL
      end
      
      def self.up
        self.setup    
        @tm.up!
      end

      def self.down
        self.setup    
        @tm.down!
      end
    end



Thanks go to the rest of the crew at SB.


Copyright (c) 2009 Serious Business, released under the MIT license
