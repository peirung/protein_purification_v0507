# Plate Transformed Cells (BL21DE3)

The objective of this protocol is to spread the transformed cell onto a LB plate with ampicilin for colonies growth.

- Workflow:
  - Heat Shock Transformation > *Plate Transformed Cells* > Check Plate
- Steps:
  - [01] Retrieve the transformed cell from 37°C shaker.
  - [02] Grab the pre-warmed LB+Amp plate.
  - [03] Spin down the transformed cell and discard the supernatant.
  - [04] Resuspend cell in 1 mL of LB.
  - [05] Use sterile beads to plate 100 µl from the transformed aliquot onto the plate. 
  - [06] Incubate the plate at 37°C overnight.
### Inputs


- **Plasmid** [P]  
  - <a href='#' onclick='easy_select("Sample Types", "Plasmid")'>Plasmid</a> / <a href='#' onclick='easy_select("Containers", "Transformed E. coli Aliquot")'>Transformed E. coli Aliquot</a>



### Outputs


- **Plate** [P]  
  - <a href='#' onclick='easy_select("Sample Types", "Plasmid")'>Plasmid</a> / <a href='#' onclick='easy_select("Containers", "E coli Plate of Plasmid")'>E coli Plate of Plasmid</a>

### Precondition <a href='#' id='precondition'>[hide]</a>
```ruby
def precondition(_op)
  true
end
```

### Protocol Code <a href='#' id='protocol'>[hide]</a>
```ruby
# Author: Ayesha Saleem
# December 23, 2016
# Modified by Pei, Oct, 2018

needs "Standard Libs/Feedback"
class Protocol
  include Feedback
  
  def name_initials str
    full_name = str.split
    begin
      cap_initials = full_name[0][0].upcase + full_name[1][0].upcase
    rescue
      cap_initials = ""
    end
    return cap_initials
  end
  
  def grab_plates plates, batch_num, ids, k
    show do
      title "Grab plates and transformed cells"
      check "Retrieve #{plates.length} pre-warmed #{k} plates from the still incubator."
      check "Label the top of the plates with your intials, the date, and the following ids: #{ids.join(", ")}"
    end
  end


  def plate_transformed_aliquots k, aliquots, plates
    show do 
      title "Plate transformed E coli aliquots"
      check "Remove the transformed cells in 1.5 mL tubes from the 250 mL flask in the 37°C incubator."
      check "Use sterile beads to plate <b>40 µl</b> from the transformed aliquots (1.5 mL tubes) onto the plates, following the table below."
      warning "Note the change in plating volume!"
      table [["1.5 mL tube", "#{k} Plate"]].concat(aliquots.zip plates)
      check "Discard used transformed aliquots after plating."
    end
  end

  

  def spin_tubes 
    op_in_plasmid = []
    operations.running.each do |op|
        op_in_plasmid << op.input("Plasmid").item.id
    end
    show do
      title "Spin down tubes and resuspend"
      note "Perform the steps with the following samples: #{op_in_plasmid.to_sentence}."
      check "Remove the transformed cells in 1.5 mL tubes from the 250 mL flask in the 37°C incubator."
    end
  end


  def group_plates_and_aliquots markers_new
    operations.each do | op | 
      p = op.input("Plasmid").item
      marker_key = "LB"
      p.sample.properties["Bacterial Marker"].split(/[+,]/).each do |marker|
        marker_key = marker_key + " + " + marker.strip[0, 3].capitalize
      end
      
      if Sample.find_by_name(marker_key)
        markers_new[marker_key][p] = op.output("Plate").item
      else
        show do 
          note "#{marker_key}"
        end
        op.error :no_marker, "There is no marker associated with this sample, so we can't plate it. Please input a marker."
      end
    end
    markers_new
  end  
  
  def calculate_and_operate markers
    markers.each do | k, v| 
      aliquots = []
      plates = []
      ids = []
        
      v.each do | al, pl|
        ids.push("#{pl.id} " + name_initials(pl.sample.user.name))
        aliquots.push(al.id)
        al.mark_as_deleted
        plates.push(pl.id)
        pl.location = "#{operations[0].input("Plasmid").sample.properties["Transformation Temperature"].to_i} C incubator"
      end
        
      b = Collection.where(object_type_id: ObjectType.find_by_name("Agar Plate Batch").id)
                        .select { |b| !b.empty? && !b.deleted? && (b.matrix[0].include? Sample.find_by_name(k).id) }.first

      if b.nil? # no agar plate batches exist
        raise "No agar plate batches for #{k} could be found in the Inventory. Pour some plates before continuing (Manager/Pour Plates)"
      end
      batch_num = [b.id]
      n = b.num_samples
      num_p = plates.length
      if n < num_p
        num_p = num_p - n
        b.apportion 10, 10
        b = Collection.where(object_type_id: ObjectType.find_by_name("Agar Plate Batch").id)
                      .select { |b| !b.empty? && !b.deleted? && (b.matrix[0].include? Sample.find_by_name(k).id) }.first
        n = b.num_samples
        batch_num.push(b.id)
      end
          
        m = b.matrix
        x = 0
    
        (0..m.length-1).reverse_each do |i|
          (0..m[i].length-1).reverse_each do |j|
            if m[i][j] != -1 && x < num_p
              m[i][j] = -1
              x += 1
            end
          end
        end
        
        # Grab and label plates
        grab_plates plates, batch_num, ids, k
        
        # Spin down tubes and resuspend
        spin_tubes
        
        # Plate transformed E. coli aliquots
        plate_transformed_aliquots k, aliquots, plates
    end
  end
    
  def main
    operations.retrieve.make
    markers_new = Hash.new { | h, k | h[k] = {} } 
    
    # group plates + transformed aliquots 
    markers = group_plates_and_aliquots markers_new
      
    # tell tech to grab x amount of plates and plate the aliquots
    # also detract from plate batches
    calculate_and_operate markers
    operations.store(io: "output", interactive: true)
    
    get_protocol_feedback
    return {}
    
  end

end
```
