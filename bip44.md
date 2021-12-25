# BIP0044
This BIP refer to the standard for using the BIP0032. Refer to original guide for official description [here](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki).

# Process
Derive the child key using the following convetion
> m / 44' / 0' / 0' / 0

Then derive number of child key

	M = m / 44' / 0' / 0' / 0
	list = []
	for i in range(num):
		m = M/i
		addr = generate_address(m)
		list.append(addr)

Generation of address is a seperate topic.