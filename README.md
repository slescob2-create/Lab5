# $s0 contains a
# $s1 contains b
# $s2 will contain the final result
# $a0 and $a1 are used to pass x and y into recursion
# $v0 is used to return the result of recursion


# Find a + b for the first recursive call
add  $t0, $s0, $s1

# Check if a + b is negative
slt  $t1, $t0, $zero
beq  $t1, $zero, abs_sum1_done

# If negative, subtract it from zero to get absolute value
sub  $t0, $zero, $t0

abs_sum1_done:

# Find a - b for the first recursive call
sub  $t2, $s0, $s1

# Check if a - b is negative
slt  $t1, $t2, $zero
beq  $t1, $zero, abs_diff1_done

# If negative, make it positive
sub  $t2, $zero, $t2

abs_diff1_done:

# Pass |a+b| as x
add  $a0, $t0, $zero

# Pass |a-b| as y
add  $a1, $t2, $zero

# Call F(|a+b|, |a-b|)
jal  recursion

# Save the returned value because $v0 will be reused
add  $s3, $v0, $zero


# Find b - a for the second recursive call
sub  $t0, $s1, $s0

# Check if b - a is negative
slt  $t1, $t0, $zero
beq  $t1, $zero, abs_diff2_done

# Make b - a positive if necessary
sub  $t0, $zero, $t0

abs_diff2_done:

# Find b + a
add  $t2, $s1, $s0

# Check if b + a is negative
slt  $t1, $t2, $zero
beq  $t1, $zero, abs_sum2_done

# Make b + a positive if necessary
sub  $t2, $zero, $t2

abs_sum2_done:

# Pass |b-a| as x
add  $a0, $t0, $zero

# Pass |b+a| as y
add  $a1, $t2, $zero

# Call F(|b-a|, |b+a|)
jal  recursion

# Save the returned value
add  $s4, $v0, $zero


# Multiply a by the first recursive result
mult $s0, $s3
mflo $t3

# Multiply b by the second recursive result
mult $s1, $s4
mflo $t4

# result = a*F1 - b*F2
sub  $s2, $t3, $t4

# Skip over the recursive procedure after main calculation finishes
j program_done


# --------------------------------------------------
# recursion
# Input:
#   $a0 = x
#   $a1 = y
#
# Output:
#   $v0 = F(x,y)
#
# A stack frame is needed because this function
# calls itself recursively. Each call must keep its
# own x, y, and return address.
# --------------------------------------------------

recursion:

# Reserve 16 bytes on the stack
# 0($sp)  stores the first recursive result
# 4($sp)  stores y
# 8($sp)  stores x
# 12($sp) stores the return address
addi $sp, $sp, -16

# Save the return address because jal changes $ra
sw   $ra, 12($sp)

# Save x because recursive calls change $a0
sw   $a0, 8($sp)

# Save y because recursive calls change $a1
sw   $a1, 4($sp)


# Test whether x is greater than zero
slt  $t0, $zero, $a0

# If x > 0, go check y
bne  $t0, $zero, x_positive


# At this point x <= 0
# Test whether y > 0
slt  $t0, $zero, $a1

# If y > 0, return y
bne  $t0, $zero, return_y

# x <= 0 and y <= 0
# F(x,y) = 0
add  $v0, $zero, $zero

j recursion_end


return_y:

# x <= 0 and y > 0
# F(x,y) = y
add  $v0, $a1, $zero

j recursion_end


x_positive:

# We already know x > 0
# Now test whether y > 0
slt  $t0, $zero, $a1

# If y <= 0, return x
beq  $t0, $zero, return_x


# Both x and y are positive, so calculate:
# F(x,y) = F(x-1,y) + F(x,y-1)


# Change x to x - 1 for the first recursive call
addi $a0, $a0, -1

# Calculate F(x-1,y)
jal  recursion

# Save F(x-1,y) in this call's stack frame
# because the next jal will change $v0
sw   $v0, 0($sp)


# Restore the original x
lw   $a0, 8($sp)

# Restore the original y
lw   $a1, 4($sp)


# Change y to y - 1 for the second recursive call
addi $a1, $a1, -1

# Calculate F(x,y-1)
jal  recursion


# Load the result of F(x-1,y)
lw   $t1, 0($sp)

# Add both recursive results together
# $v0 currently contains F(x,y-1)
add  $v0, $t1, $v0

j recursion_end


return_x:

# x > 0 and y <= 0
# F(x,y) = x
add  $v0, $a0, $zero


recursion_end:

# Restore the original y
lw   $a1, 4($sp)

# Restore the original x
lw   $a0, 8($sp)

# Restore the return address for this recursive call
lw   $ra, 12($sp)

# Remove this call's stack frame
addi $sp, $sp, 16

# Return to the caller
jr   $ra


program_done:
