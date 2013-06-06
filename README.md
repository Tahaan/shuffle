shuffle
=======

An encryption algorithm inspired by the desire to disprove some classical encryption principles:

Single-key encryption is bad/useless
Single-key encryption becomes increasingly easier to break by brute force when the source data complies to basic rules, such as english text
Single-key encryption becomes increasingly easier to break when the amount of data encrypted with one key becomes bigger (Eg beat pattern matching techniques)
Better encryption is expensive in terms of CPU
Better encryption is mathematically complicated

The a-ha moment was when I was looking at a children's game of shuffling letters to find the hidden word ... but what if you could not see what each letter was.

The algorithm requires the following

1. For ENCRYPTION input is:
a) a block of plain-text data (characters/bytes) of 254 or less bytes,
b) a secret key consisting of an array of between 2 and 255 bytes.

Output is a block of 256 bytes of cypher-text data plus optionally an MD5 checksum.

2. For DECRYPTION input is:
a) a block of encrypted data (characters/bytes) of a 255 cypher-text bytes
b) a secret key consisting of an array of between 2 and 255 bytes.
c) an optional MD5 sum of the encrypted data.  This feature though implementation specific is recommended.

Output is a block of up to 254 bytes of plain-text data

  The overall procedure for Encryption is:

1. Optional: Set up the Translocation map using the secret key.
2. Repeat while there is more data to encrypt
  2.1 Read a block of up to 255 bytes of plain-text data from the input file.
    2.1.1 Insert the length of the plain-text at index=255 of the plain text bringing the total length to 256 bytes.
    2.1.2 If the read plain-text is less than 255 bytes in length (256 including length), pad with random bytes to bring total length to 255 bytes.
  2.2. Apply the Encryption algorithm to the entire block to obtain the cipher-text block.
  2.3. Calculate the checksum of the cypher-text and append to the cipher-text block for output
  2.4. Write the Encryption Header, output cipher-text, and Checksum to the output
Repeat step 2 till End of input file.

The header may include an implementation/compatibility marker indiciting variations such as omission of the MD5 check-sum and possibly using a default block size other than 256 bytes and a length marker other than 1 byte.

Handling of plain text and binary data
* Plain text is assumed to by "binary data".  Each ASCII character is processed using its implicit byte conversion value.
* A secret key can be directly derived by using implicit ASCII to byte conversion of the characters in the password string.

  Encrypting the plain-text:
The data is processed one byte at a time.  Each byte is first transposed and then translocated and the resulting cypher-data written to the output array.

Transposition consists of a character "Rotated" by a value depending on its position in the input data and the secret key.  The first character of the secret key is used to process the first character of the plain-text, the second character of the secret key is used to process the second character of the plain-text, and so on until either the secret key runs out or all the characters in the plain text block, including length and padding characters, have been processed.  If the secret key runs out, start over from the first character of the key array.

A character is processed by adding to its byte value the byte value from the corresponding character in the secret key.  Resulting values larger than 255 gets 255 subtracted.

This transposed character is then written to a location in the cipher-data which is determined by consulting the translocation map.  Each byte in the transloction map specifies the position (index) where the transposed character is to be stored in the cypher-data block.  The array index value of the translocation map values is used from the input data index, ie the first character of input data is translocated using the first value in the translocation map, the second character by using the second value in the translocation map, and so on.

  Setting up the Translocation Map:
For each character read its target location in the output data is *incrementally* based on the corresponding character in the secret key array.  The first character to be encrypted will be written to the output array at index = ASCII value of the first character of the secret key. Incrementally each subsequent character is offset from the last character, so that each subsequent character is written at index = previous index + ASCII value of next character in the secret key.

MAP [1] = KEY[1]
MAP [2] = MAP[1]+KEY [2]
MAP [n] = MAP[n-1]+KEY[n]

Cipher(INPUT [1]) -> OUTPUT [MAP[1]]
Cipher(INPUT [2]) -> OUTPUT [MAP[2]]
Cipher(INPUT [n]) -> OUTPUT [MAP[n]]

In calculating the map when the index goes beyond the size of the output block, the index wraps around.  In determining the next available slot in the output, any already used slots are not counted.

Example using indexes starting from 1 and a short (5-byte) data block and 5-byte secret key.

So with a secret key = 2, 5, 3, 4, 5
Key index              1  2  3  4  5
MAP                    2, 3, 1, 5, 4

Building the Map
After Step 1           _  X  _  _  _
After Step 2           _  X  X  _  _
After Step 3           X  X  X  _  _
After Step 4           X  X  X  _  X
After Step 5           X  X  X  X  X

Note: This changes when 1 character is added to the last position for the clear text length.

The first character goes to OUTPUT [2], eg MAP [1]=2
The second character goes to OUTPUT [2+5] = 7.  Since 7 is >=5, it wraps arround to 7-5=2.  But since 2 is already used, it steps over to the next slot which is 3.  Thus MAP[2] = 3
The next character goes to OUTPUT [1] (eg MAP[3] = 1) because MAP[3-1]+KEY[3] - 5 = 3 + 3 - 5 = 1
MAP [3] will step over slot 2 and 3 since these are used already, and start counting from 4 as the first available slot.  5 is the only other available slot, so it counts 4, 5, skip, skip, skip, 4, 5.
Finally for the last character there is only one possible outcome.  Generating the Translocation Map is optimized through using an allocation/free slots map.

Using the resulting KEY and MAP above and using the word "hello" as an example plain-text, the results is
Cipher(Input[n]) -> Output[MAP[n]]

For decrypting
Decypher(Input[MAP[n]) -> Output[n]

Cypher(n) is simply
If n > Length(Key)
   X = n mod Length(key)
else
   X = n
If (Key[X] + Input[X] > 255)
   return (Key[n] + Input[X] - 256)
else
   return (Key[n] + Input[X]

Decypher(n) is similar:
If n > Length(Key)
   X = n mod Length(key)
else
   X = n
If (Input[Map[n]) - Key[X] < 0)
   return 256+Input[Map[n]]-Key[X]
else
   return Input[Map[n]]-Key[X]

Examples without 

INPUT = h, e, l, l, o, 5
KEY   = 2, 5, 3, 4, 5
Cipher= j, j, o, p, t
MAP   = 2, 3, 1, 5, 4
OUTPUT= o, j, j, t, p

The important thing to realize is that when one bit of the secret key changes, not only does the output change, but the order in which the characters are written also changes.

INPUT = h, e, l, l, o
KEY   = 3, 5, 3, 4, 5
Cipher= k, j, o, p, t
MAP   = 3, 4, 2, 1, 5
OUTPUT= p, o, k, j, t

